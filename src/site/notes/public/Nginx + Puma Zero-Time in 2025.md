---
{"dg-publish":true,"permalink":"/public/nginx-puma-zero-time-in-2025/","title":"Nginx + Puma Zero-Time in 2025"}
---


`show-frontmatter-title: true`

This document outlines the necessary steps and configuration changes for NGINX, Puma, and Capistrano to enable zero-downtime deployments.

The official Puma documentation and guidelines (as of 2025) are not entirely clear or complete when it comes to zero-downtime deployments. The steps in this guide reflect the most reliable and up-to-date approach, based on real-world debugging of a production application and insights gathered from recent GitHub issues and community discussions.

## Set fail_timeout=0 for the upstream

Without `fail_timeout=0`, if Puma is temporarily unavailable (e.g., during a phased restart), Nginx will mark the backend as failed and avoid sending requests to it for the default timeout period (e.g., 10s). This can result in 502 errors during deployments. Setting `fail_timeout=0` ensures NGINX retries the backend immediately on the next request.

```nginx
upstream app {
  server unix:{{ app_base_path }}/shared/tmp/sockets/puma.sock fail_timeout=0;
}
```

## Send USR1 to gracefully restart Puma

To enable phased (zero-downtime) restarts of Puma via `systemctl reload puma`, we need to update Puma’s systemd unit by specifying the ExecReload directive. This directive should send the USR1 signal to the Puma master process, allowing it to restart workers without dropping active connections.

```ini
[Service]

# ...some service settings here...

ExecReload=/bin/kill -USR1 $MAINPID

```

## Properly configure puma for phased-restart

One critical but non-obvious requirement is to explicitly set `preload_app!` to false. By default, `preload_app!` is enabled, and using `fork_worker` does not override this. However, phased restarts are not compatible with `preload_app! true` and will not function correctly unless it is disabled.

This is because with `preload_app! true`, the application code is loaded into memory in the master process before forking workers. As a result, newly forked workers will inherit the already-loaded (and potentially outdated) code, making phased restarts ineffective.

```ruby
preload_app! false
prune_bundler
fork_worker
```

The next important step is to ensure that Active Record re-establishes the database connection in each worker after it is forked. This is necessary because database connections do not survive across process forks — each worker must open its own connection to avoid using a stale or broken one.

```ruby
on_worker_boot do
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
end
```

We also need to activate Puma’s state file (e.g., PID, control socket) and control app, which are required for phased restarts and control commands like pumactl stats or phased-restart. These features allow Puma to manage worker lifecycle and respond to signals without downtime.

```ruby
state_path "#{shared_dir}/pids/puma.state"
activate_control_app "unix://#{shared_dir}/sockets/pumactl.sock"
```

Last but not least, it’s important to properly set the directory option so that Puma changes its working directory after Capistrano symlinks the new release. Without this, Puma will continue to load code from the original release directory where it was first started, which breaks phased restarts and prevents new code from being picked up.

```ruby
directory "#{File.expand_path('../../..', __dir__)}/current"
```

## Update capistrano tasks

Please note that you should call `invoke("puma:reload")` instead of `invoke("puma:restart")` to ensure zero-downtime deployments. A hard restart (`puma:restart`) should only be triggered manually when necessary, for example by running `cap production puma:restart`. Using reload allows Puma to perform a phased restart and preserve active connections.

```ruby
namespace :deploy do
  %w[start stop restart].each do |action|
    desc "#{action.capitalize} application"
    task action.to_sym do
      on roles(:web) do
        if action == 'restart'
          invoke("puma:reload")
        else
          invoke("puma:#{action}")
        end
        
  # ...some code...  
```

## Capistrano task for checking Puma status

Sometimes it’s helpful to inspect the current state of the Puma server — including worker status, thread usage, request backlog, and more. Since we’ve enabled the control app and Puma state file, we can easily add a Capistrano task to gather and display this runtime information.

```ruby
desc 'Show Puma thread usage including backlog'
task :stats do
  on roles(:web) do
    within current_path do
      state_file = "#{shared_path}/tmp/pids/puma.state"

      output = capture(:bundle, :exec, :pumactl, "-S", state_file, "stats")
      json_str = output.lines[1..].join
      stats = JSON.parse(json_str)

      puts "\n💡 Per-worker thread usage:"
      stats["worker_status"].each do |worker|
        index    = worker["index"]
        status   = worker["last_status"]
        running  = status["running"]
        max      = status["max_threads"]
        backlog  = status["backlog"]
        capacity = status["pool_capacity"]
        count    = status["requests_count"]
        oldest   = status["oldest_request_start"] || "—"

        puts "  • Worker #{index}: #{running}/#{max} threads used | backlog: #{backlog} | pool: #{capacity} | requests: #{count} | oldest: #{oldest}"
      end

      total_running = stats["worker_status"].sum { |w| w.dig("last_status", "running").to_i }
      total_max     = stats["worker_status"].sum { |w| w.dig("last_status", "max_threads").to_i }
      total_backlog = stats["worker_status"].sum { |w| w.dig("last_status", "backlog").to_i }

      puts "\n📈 Total: #{total_running}/#{total_max} threads used | total backlog: #{total_backlog}"
    end
  end
end
```

## Check if phased-restart works

To verify that phased restarts are working, open the Puma log using journalctl -u puma -f, then run systemctl reload puma to send the USR1 signal and initiate a phased restart. In the logs, you should see a message similar to the one below — where phase: 3 indicates the number of zero-downtime restarts that have occurred since Puma was started (i.e., this is the third phased restart).

![puma-phased-restart-log.png.png|600](/img/user/public/data/puma-phased-restart-log.png.png)
