require 'rubygems'
require 'puppet'
require 'colored'
require 'rake/clean'
require 'puppet/network/http_pool'
require 'uri'
require 'puppet/util/puppetdb'
require 'yaml'

desc "verifies correctness of node syntax"
task :verify_nodes, [:manifest_path, :module_path, :nodename_filter, :class_filter] do |task, args|
  fail "manifest_path must be specified" unless args[:manifest_path]
  fail "module_path must be specified" unless args[:module_path]

  setup_puppet args[:manifest_path], args[:module_path]
  nodes = collect_puppet_nodes args[:nodename_filter], args[:class_filter]
  failed_nodes = {}
  puts "Found: #{nodes.length} nodes to evaluate".cyan
  nodes.each do |nodename|
    print "Verifying node #{nodename}: ".cyan
    begin
      compile_catalog(nodename)
      puts "[ok]".green
    rescue => error
      puts "[FAILED] - #{error.message}".red
      failed_nodes[nodename] = error.message
    end
  end
  puts "The following nodes failed to compile => #{print_hash failed_nodes}".red unless failed_nodes.empty?
  raise "[Compilation Failure] at least one node failed to compile" unless failed_nodes.empty?
end

def print_hash nodes
  nodes.inject("\n") { |printed_hash, (key,value)| printed_hash << "\t #{key} => #{value} \n" }
end

def puppetdb_facts(nodename)
  conn = Puppet::Network::HttpPool.http_instance(Puppet::Util::Puppetdb.server, Puppet::Util::Puppetdb.port, use_ssl = true)
  response = conn.get("/facts/#{nodename}", { "Accept" => "application/json",})
  raise Puppet::ParseError, "PuppetDB query error: [#{response.code}] #{response.msg}" unless response.kind_of?(Net::HTTPSuccess)
  PSON.load(response.body)['facts']
end

def puppetdb_nodes()
  conn = Puppet::Network::HttpPool.http_instance(Puppet::Util::Puppetdb.server, Puppet::Util::Puppetdb.port, use_ssl = true)
  params = URI.escape("?query=[\"=\", [\"node\", \"active\"], true ]")
  response = conn.get("/nodes#{params}", { "Accept" => "application/json",})
  raise Puppet::ParseError, "PuppetDB query error: [#{response.code}] #{response.msg}" unless response.kind_of?(Net::HTTPSuccess)
  PSON.load(response.body)
end

def enc(nodename)
  YAML.load(`#{Puppet[:external_nodes]} #{nodename}`)
end

def compile_catalog(nodename)
  facts = puppetdb_facts(nodename)
  encinfo = enc(nodename)
  node = Puppet::Node.new(nodename, {:classes => encinfo['classes'],
                                     :environment => 'production',
                                     :parameters => encinfo['parameters'] })
  node.merge(facts)
  Puppet::Parser::Compiler.compile(node)
end

def collect_puppet_nodes(filter = ".*", classfilter = nil)
  nodes = puppetdb_nodes()
  nodes.select { |node| node =~ /#{filter}/ }
end

def setup_puppet manifest_path, module_path
  Puppet.settings.handlearg("--config", "./puppet.conf")
  Puppet.settings.handlearg("--manifest", manifest_path)
  Puppet.settings.handlearg("--modulepath", module_path)
  Puppet.parse_config
end
