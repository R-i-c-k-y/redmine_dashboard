#
#

BASE = Dir.getwd

def version; ENV["REDMINE_VERSION"] || "2.3.2" end
def archive; "#{version}.tar.gz" end

def exec(*attrs)
  puts "+ #{attrs.join(' ')}"
  unless system(*attrs)
    raise StandardError.new "Command failed: #{attrs.join(' ')}"
  end
end

def unless_done(name, version = 1, &block)
  donefile = "#{path}/.rdb-done-#{name}.#{version}"
  envs     = []
  if File.exists? donefile
    envs = IO.read(donefile).split("\n").map{|s|s.strip}
  end
  unless envs.include? RUBY_DESCRIPTION
    yield
    exec "echo '#{RUBY_DESCRIPTION}' >> #{donefile}"
  end
end

def tmpdir; "#{BASE}/tmp" end
def archives_path; "#{tmpdir}/archives" end
def redmines_path; "#{tmpdir}/redmines" end
def databases_path; "#{tmpdir}/databases" end
def database_path; "#{databases_path}/#{version}" end
def archive_path; "#{archives_path}/#{archive}" end
def path; "#{redmines_path}/#{version}" end
def bx(*attrs); exec *(['bundle', 'exec'] + attrs) end
def bxrake(*attrs); bx *(['rake'] + attrs) end
def mkpath(*paths); exec *(['mkdir', '-p'] + paths) end

namespace :redmine do
  desc 'Install redmine to tmp dir.'
  task :install => [ :download ] do
    unless_done('install', 1) do
      Dir.chdir path do
        # database config
        mkpath database_path
        File.open("#{path}/config/database.yml", 'w') do |file|
          file.write <<-DATABASE
          common: &common
            adapter: sqlite3
            pool: 5
            timeout: 5000
          test:
            <<: *common
            database: #{database_path}/test.sqlite3
          production:
            <<: *common
            database: #{database_path}/production.sqlite3
          development:
            <<: *common
            database: #{database_path}/development.sqlite3
          DATABASE
        end

        # symlink plugin
        exec 'rm', "#{path}/plugins/redmine_dashboard" rescue true
        exec 'ln', '-s', BASE, "#{path}/plugins/"

        # symlink assets for development mode
        mkpath "#{path}/public/plugin_assets"
        exec 'rm', "#{path}/public/plugin_assets/redmine_dashboard_linked" rescue true
        exec 'ln', '-s', "#{BASE}/assets", "#{path}/public/plugin_assets/redmine_dashboard_linked"

        # install dependencies
        exec 'bundle', 'install'
      end
      puts
    end
  end

  desc 'Setup and initialize redmine.'
  task :setup => [ :install ] do
    unless_done('setup', 1) do
      Dir.chdir path do
        bxrake 'db:migrate'
        bxrake 'redmine:load_default_data', 'REDMINE_LANG=en'
        bxrake 'generate_secret_token'
        bxrake 'redmine:plugins:migrate'
      end
      puts
    end
  end

  task :download do
    unless File.exists? "#{path}/Gemfile"
      unless File.exists? archive_path
        mkpath archives_path
        Dir.chdir archives_path do
          exec 'wget', "https://github.com/edavis10/redmine/archive/#{archive}"
        end
      end
      if File.exists? archive_path
        mkpath path
        exec 'tar', '-C', path, '-xz', '--strip=1', '-f', archive_path
      end
    end
  end

  desc 'Remove installed redmine'
  task :remove do
    exec 'rm', '-rf', path
  end

  desc 'Run local redmine development server'
  task :server => [ :setup ] do
    Dir.chdir path do
      bx 'rails', 'server'
    end
    puts
  end
end

desc 'Run redmine_dashboard specs.'
task :spec => [ 'redmine:setup' ] do
  Dir.chdir path do
    bx 'rspec', 'plugins/redmine_dashboard/spec'
  end
end

task :default => :spec
