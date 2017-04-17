# (c) 2016 DataNexus Inc.  All Rights Reserved
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

# monkey-patch that is used to leave unrecognized options in the ARGV
# list so that they can be processed by underlying vagrant command
class OptionParser
  # Like order!, but leave any unrecognized --switches alone
  def order_recognized!(args)
    extra_opts = []
    begin
      order!(args) { |a| extra_opts << a }
    rescue OptionParser::InvalidOption => e
      extra_opts << e.args[0]
      retry
    end
    args[0, 0] = extra_opts
  end
end

# initialize a few values
options = {}

# vagrant commands that include these commands can be run without specifying
# any IP addresses or without including the application name
no_ip_commands = ['version', 'global-status', '--help', '-h']
no_name_commands = ['status', 'ssh', 'destroy']

# define the options we're adding to the vagrant command
optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:app_name] = nil
  opts.on( '-n', '--name NAME', 'Name of the application being deployed' ) do |app_name|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-n=rstudio')
    options[:app_name] = app_name.gsub(/^=/,'')
  end

  options[:node_addr] = nil
  opts.on( '-a', '--addr IP_ADDR', 'IP address of the node' ) do |node_addr|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-a=192.168.1.1')
    options[:node_addr] = node_addr.gsub(/^=/,'')
  end

  options[:app_url] = nil
  opts.on( '-u', '--url APP_URL', 'URL for application distribution package' ) do |app_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:app_url] = app_url.gsub(/^=/,'')
  end

  options[:cran_repo_url] = nil
  opts.on( '-r', '--cran-url CRAN_URL', 'URL of CRAN mirror' ) do |cran_repo_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-c=http://localhost/tmp.tgz')
    options[:cran_repo_url] = cran_repo_url.gsub(/^=/,'')
  end

  options[:local_app_file] = nil
  opts.on( '-l', '--local-app-file File', 'Local application distribution file (package)' ) do |local_app_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=/tmp/tmp.tgz')
    options[:local_app_file] = local_app_file.gsub(/^=/,'')
  end

  options[:yum_repo_url] = nil
  opts.on( '-y', '--yum-url URL', 'Local yum repository URL' ) do |yum_repo_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-y=http://192.168.1.128/centos')
    options[:yum_repo_url] = yum_repo_url.gsub(/^=/,'')
  end

  options[:epel_repo_url] = nil
  opts.on( '-e', '--epel-url URL', 'Local epel repository URL' ) do |epel_repo_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-e=http://192.168.1.128/epel')
    options[:epel_repo_url] = epel_repo_url.gsub(/^=/,'')
  end

  options[:reset_proxy_settings] = false
  opts.on( '-c', '--clear-proxy-settings', 'Clear existing proxy settings if no proxy is set' ) do |reset_proxy_settings|
    options[:reset_proxy_settings] = true
  end

  options[:reset_undef_mirrors] = false
  opts.on( '-d', '--use-default-mirrors', 'Reset YUM/EPEL mirror definitions (if not defined)' ) do |reset_undef_mirrors|
    options[:use_default_mirrors] = true
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

# parse the command-line options that were passed in
begin
  optparse.order_recognized!(ARGV)
rescue SystemExit => e
  exit
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

# check remaining arguments to see if the command requires
# an IP address (or not)
ip_required = (ARGV & no_ip_commands).empty?
name_required = (ARGV & (no_name_commands + no_ip_commands)).empty?

# and check the values passed in by the user on the command-line
if name_required && !options[:app_name]
  print "ERROR; application name must be specified for vagrant commands\n"
  print optparse
  exit 1
end

if ip_required && !options[:node_addr]
  print "ERROR; server IP address must be supplied for vagrant commands\n"
  print optparse
  exit 1
elsif options[:node_addr] && !(options[:node_addr] =~ Resolv::IPv4::Regex)
  print "ERROR; input server IP address '#{options[:node_addr]}' is not a valid IP address"
  exit 2
end

if options[:app_url] && options[:local_app_file]
  print "ERROR; the app URL and local app file options cannot be used simultaneously"
end

if options[:app_url] && !(options[:app_url] =~ URI::regexp)
  print "ERROR; input app URL '#{options[:app_url]}' is not a valid URL\n"
  exit 3
end

if options[:epel_repo_url] && !(options[:epel_repo_url] =~ URI::regexp)
  print "ERROR; input epel URL '#{options[:epel_repo_url]}' is not a valid URL\n"
  exit 3
end

if options[:cran_repo_url] && !(options[:cran_repo_url] =~ URI::regexp)
  print "ERROR; input app URL '#{options[:cran_repo_url]}' is not a valid URL\n"
  exit 3
end

if options[:local_app_file] && !File.exists?(options[:local_app_file])
  print "ERROR; input local app file '#{options[:local_app_file]}' is not a local file\n"
  exit 3
end

# if a yum repository address was passed in, check and make sure it's a valid URL
if options[:yum_repo_url] && !(options[:yum_repo_url] =~ URI::regexp)
  print "ERROR; input yum repository URL '#{options[:yum_repo_url]}' is not a valid URL\n"
  exit 6
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
node_addr_array = options[:node_addr].split(',') if options[:node_addr]
if node_addr_array && node_addr_array.size > 0
  Vagrant.configure("2") do |config|
    proxy = ENV['http_proxy'] || ""
    no_proxy = ENV['no_proxy'] || ""
    proxy_username = ENV['proxy_username'] || ""
    proxy_password = ENV['proxy_password'] || ""
    if Vagrant.has_plugin?("vagrant-proxyconf")
      if $proxy
        config.proxy.http               = $proxy
        config.vm.box_download_insecure = true
        config.vm.box_check_update      = false
      end
      if $no_proxy
        config.proxy.no_proxy           = $no_proxy
      end
      if $proxy_username
        config.proxy.proxy_username     = $proxy_username
      end
      if $proxy_password
        config.proxy.proxy_password     = $proxy_password
      end
    end
    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://atlas.hashicorp.com/search.
    config.vm.box = "centos/7"
    config.vm.box_check_update = false
    # set the memory for the box to 2GB
    config.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
    end
    # loop through the node address array and create a VM for each
    node_addr_array.each do |machine_addr|
      config.vm.define machine_addr do |machine|
        # setup a private network for this machine
        config.vm.network "private_network", ip: machine_addr
        # if this is the last machine in the list, then provision all of the
        # machines simultaneously
        if machine_addr == node_addr_array[-1]
          # use the playbook in the `site.yml' file to provision app to our node
          config.vm.provision "ansible" do |ansible|
            # set the limit to 'all' in order to provision all of machines on the
            # list in a single playbook run
            ansible.limit = "all"
            ansible.playbook = "site.yml"
            ansible.groups = {
              "#{options[:app_name]}": node_addr_array
            }
            ansible.extra_vars = {
              proxy_env: {
                http_proxy: proxy,
                no_proxy: no_proxy,
                proxy_username: proxy_username,
                proxy_password: proxy_password
              },
              application: options[:app_name],
              yum_repo_url: options[:yum_repo_url],
              local_app_file: options[:local_app_file],
              host_inventory: node_addr_array,
              reset_proxy_settings: options[:reset_proxy_settings],
              inventory_type: "static"
            }
            # set the `rstudio_url` if a value for that parameter was included on the command-line
            if options[:rstudio_url]
              ansible.extra_vars[:app_url] = options[:rstudio_url]
            end
            # set the `cran_repo_url` if a value for that parameter was included on the command-line
            if options[:cran_repo_url]
              ansible.extra_vars[:cran_repo_url] = options[:cran_repo_url]
            end
            # set the `epel_repo_url` if a value for that parameter was included on the command-line
            if options[:epel_repo_url]
              ansible.extra_vars[:epel_repo_url] = options[:epel_repo_url]
            end
            if options[:reset_undef_mirrors]
              ansible.extra_vars[:reset_undef_mirrors] = true
            end
          end     # end `machine.vm.provision "ansible" do |ansible|`
        end
      end
    end
  end     # end `Vagrant.configure ("2") do |config|`
end
