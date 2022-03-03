uname
#!/usr/bin/env ruby
require "json"

VAGRANTFILE = File.expand_path("~/Projects/.vagrant")
USE_BUNDLER = false
VM_ID = JSON.load(File.read(VAGRANTFILE))["active"]["default"]

def find_working_dir(file)
  working_dir = File.dirname(file)

  while true
    exists = ["spec", "test", "lib", "Gemfile", "node_modules"].find do |dir|
      File.exist?("#{working_dir}/#{dir}") && working_dir !~ %r[/(spec|test)/]
    end

    return working_dir if exists
    return if working_dir == "/"

    working_dir = File.dirname(working_dir)
  end
end

if ARGV.size == 2
  command, file = *ARGV
else
  fail "Invalid builder parameters"
end

working_dir = find_working_dir(file)

if USE_BUNDLER && working_dir && File.exist?("#{working_dir}/Gemfile")
  bundle_exec = "bundle exec"
end

full_command = ""
full_command << "cd '#{working_dir}' && " if working_dir
full_command << "#{bundle_exec} #{command} '#{file}'"

system <<-SH
  VBoxManage guestcontrol #{VM_ID} \
    execute --image /bin/sh \
    --username vagrant --password 'vagrant' \
    --environment 'HOME=/home/vagrant PATH=./bin:/home/vagrant/local/bin:/home/vagrant/local/ruby/gems/bin:/usr/local/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin GEM_PATH=/home/vagrant/local/ruby/gems GEM_HOME=/home/vagrant/local/ruby/gems' \
    --wait-stdout --wait-stderr --wait-exit \
    -- -c "#{full_command}"
