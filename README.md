# Introduction
Typically, Chef is used to configure a node and something like Capistrano is used to deploy code. In this solution, Chef is used to both bootstrap the node and deploy a docker image. Subsequent chef-client runs will deploy the latest application docker image with zero downtime.

# Strategic Overview
A front facing native nginx web server accepts http requests on port 80 and forwards these requests upstream to a docker container listening on either port 80 or 81. Chef will create a sibling release container on the alternate port (either 81 or 82 depending on which is available) and then gracefully restart the front facing nginx web server to forward requests to the new docker container.

![Chef logic overview](http://i.imgur.com/68j1zmr.png)

# Four steps to Chef logic.

```
# 1. Initial server configuration and setup

include_recipe 'apt::default'

apt_repository 'nginx' do
  uri          'http://nginx.org/packages/ubuntu/'
  distribution 'xenial'
  components   ['nginx']
  key          'http://nginx.org/keys/nginx_signing.key'
  deb_src      true
end

apt_package 'nginx' do
  action :install
  notifies :run, 'execute[nginx startup fix]', :immediately
end

execute 'nginx startup fix' do
  command "ps -ef | grep nginx | grep -v grep | awk '{print $2}' | xargs sudo kill -9"
  command "/usr/sbin/nginx -c /etc/nginx/nginx.conf &"
  action :nothing
end

docker_installation_script 'default' do
  repo 'main'
  script_url 'https://get.docker.com/'
  action :create
end

# 2. Create sibling 'release' container

ruby_block 'discover port used by the current container' do
   block do
     Chef::Resource::RubyBlock.send(:include, Chef::Mixin::ShellOut)
     command = 'sudo docker port current_container 80'
     output = shell_out(command).stdout.to_s
     node.default[:current_port] = output[/#{"0.0.0.0:"}(.*?)#{"\n"}/m, 1]
   end
 end

 ruby_block 'select alternate port for new release container' do
   block do
     node.default[:release_port] = node.default[:current_port].eql?("81") ? '82' : '81'
   end
 end

 docker_image 'zimchi/testapp' do #use this image or any other on dockerhub
   tag 'latest'
   action :pull
 end

 ruby_block 'create release container' do
   block do
     Chef::Resource::RubyBlock.send(:include, Chef::Mixin::ShellOut)
     command = "sudo docker run -d --name release_container -p #{node.default[:release_port]}:80 zimchi/testapp:latest"
     output = shell_out(command).stdout.to_s
   end
 end

# 3. Update Nginx for new container

  template '/etc/nginx/nginx.conf' do #see template for details
    source 'nginx.conf.erb'
    owner 'root'
    group 'root'
    mode '0644'
    action :create
  end

  execute 'gracefully restart nginx' do
    command "sudo kill -s HUP #{`cat /var/run/nginx.pid`}"
  end

# 4. Clean up

  ruby_block 'delete old current_container' do
    block do
      command = 'sudo docker stop current_container && sudo docker rm current_container'
      shell_out(command).stdout.to_s
    end
  end

  ruby_block 'rename release_container to current_container' do
    block do
      command = 'sudo docker rename release_container current_container'
      shell_out(command).stdout.to_s
    end
  end

```
# Note
  * Assumes password-less sudo on your server for docker commands (default on AWS)
  * Chef-client must be run a second time after provisioning a fresh node (known bug)
  * Test success using /test-app.txt
