require "bundler/gem_tasks"
require 'rspec/core/rake_task'
require 'tempfile'
require 'net/ssh'

desc 'Run unit and integration specs.'
task :spec => ['spec:unit', 'spec:integration:all']

namespace :spec do
  RSpec::Core::RakeTask.new("unit") do |task|
    task.ruby_opts = '-I ./spec/unit'
    task.pattern = "./spec/unit{,/*/**}/*_spec.rb"
  end

  namespace :integration do
    targets = ["ubuntu:trusty"]
    container_name = 'itamae'
    suites = [
      [
        "spec/integration/recipes/default.rb",
        "spec/integration/recipes/default2.rb",
        "spec/integration/recipes/redefine.rb",
      ],
      [
        "--dry-run",
        "spec/integration/recipes/dry_run.rb",
      ],
    ]

    task :all     => targets.flat_map { |target| ["docker:#{target}", "local:#{target}"] }
    targets.each do |target|
      namespace :docker do
        namespace :docker_run do
          desc "Run docker for #{target}"
          task target do
            sh "docker run --privileged -d --name #{container_name} #{target} /sbin/init"
          end
        end

        desc "Run provision and specs to #{target} with `itamae docker` command"
        task target => ["docker_run:#{target}", "docker:provision:#{target}", "docker:serverspec:#{target}", 'clean_docker_container']

        namespace :provision do
          desc "Run itamae to #{target}"
          task target do
            suites.each do |suite|
              cmd = %w!bundle exec bin/itamae docker!
              cmd << "-l" << (ENV['LOG_LEVEL'] || 'debug')
              cmd << "-j" << "spec/integration/recipes/node.json"
              cmd << "--container" << container_name
              cmd << "--tag" << "itamae:latest"
              cmd += suite

              p cmd
              unless system(*cmd)
                raise "#{cmd} failed"
              end
            end
          end
        end

        namespace :serverspec do
          desc "Run serverspec tests to #{target}"
          RSpec::Core::RakeTask.new(target.to_sym) do |t|
            ENV['DOCKER_CONTAINER'] = container_name
            t.ruby_opts = '-I ./spec/integration'
            t.pattern = "spec/integration/*_spec.rb"
          end
        end
      end

      namespace :local do
        desc "Run provision and specs to #{target} with `itamae local` command"
        task target => ["docker_run:#{target}", "local:provision:#{target}", "local:serverspec:#{target}", 'clean_docker_container']

        namespace :docker_run do
          desc "Run docker for #{target}"
          task target do
            sh "docker run --privileged -d --name #{container_name} -v $(pwd):/itamae #{target} /sbin/init"
          end
        end

        namespace :provision do
          desc "Run itamae to #{target}"
          task target do
            suites.each do |suite|
              cmd = %w!bundle exec bin/itamae docker!
              cmd << "-l" << (ENV['LOG_LEVEL'] || 'debug')
              cmd << "-j" << "spec/integration/recipes/node.json"
              cmd << "--container" << container_name
              cmd << "--tag" << "itamae:latest"
              cmd += suite

              p cmd
              unless system(*cmd)
                raise "#{cmd} failed"
              end
            end
          end
        end

        namespace :serverspec do
          desc "Run serverspec tests to #{target}"
          RSpec::Core::RakeTask.new(target.to_sym) do |t|
            ENV['DOCKER_CONTAINER'] = container_name
            t.ruby_opts = '-I ./spec/integration'
            t.pattern = "spec/integration/*_spec.rb"
          end
        end
      end
    end

    desc 'Clean a docker container for test'
    task :clean_docker_container do
      sh('docker', 'rm', '-f', container_name)
    end
  end
end

