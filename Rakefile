require "erb"

STDOUT.sync = true
BUILDER_BASE = File.dirname(__FILE__)
BUILDER_TEMPLATES = File.join(BUILDER_BASE, "templates")
BUILDER_PLUGIN_SOURCE = File.join(BUILDER_BASE, "plugins")

COLLECTIVEDIR = File.join(BUILDER_BASE, "collective")
COLLECTIVE_SSL = File.join(COLLECTIVEDIR, "ssl")
COLLECTIVE_BASE = File.join(COLLECTIVEDIR, "base")
COLLECTIVE_CHORIA = File.join(COLLECTIVEDIR, "choria")
COLLECTIVE_PIDS = File.join(COLLECTIVEDIR, "pid")
COLLECTIVE_LOGS = File.join(BUILDER_BASE, "logs")
COLLECTIVE_CLIENT = File.join(COLLECTIVEDIR, "client.choria")

def ask(prompt, env, default="")
  return ENV[env] if ENV.include?(env)

  print "%s (%s): " % [prompt, default]
  resp = STDIN.gets.chomp

  resp.empty? ? default : resp
end

def render_template(template, output, scope)
  tmpl = File.read(template)
  erb = ERB.new(tmpl, 0, "<>")
  File.open(output, "w") do |f|
    f.puts erb.result(scope)
  end
end

def ca_path
  File.join(COLLECTIVE_SSL, "certs", "ca.pem")
end

def cert_path(certname)
  File.join(COLLECTIVE_SSL, "certs", "%s.pem" % certname)
end

def private_key_path(certname)
  File.join(COLLECTIVE_SSL, "private_keys", "%s.pem" % certname)
end

def has_cert?(certname)
  File.exist?(cert_path(certname)) && File.exist?(private_key_path(certname))
end

def gen_cert(certname)
  return if has_cert?(certname)

  puts "Creating SSL certificate for %s" % certname

  %x[puppet cert --ssldir #{COLLECTIVE_SSL} generate #{certname}]
end

def setup_base(gitrepo, branch)
  FileUtils.mkdir_p(COLLECTIVE_BASE)
  FileUtils.mkdir_p(COLLECTIVE_PIDS)
  FileUtils.mkdir_p(COLLECTIVE_LOGS)
  FileUtils.mkdir_p(COLLECTIVE_CLIENT)

  puts "Checking out MCollective git repository into %s" % [COLLECTIVE_BASE]

  system("git clone %s %s --branch %s" % [gitrepo, COLLECTIVE_BASE, branch])
end

def render_member_config(identity)
  instance_home = member_dir(identity)

  puts "Rendering configuration files for instance %s" % identity

  render_template(File.join(BUILDER_TEMPLATES, "server.cfg.erb"), File.join(instance_home, "etc", "server.cfg"), binding)
  render_template(File.join(BUILDER_TEMPLATES, "client.cfg.erb"), File.join(instance_home, "etc", "client.cfg"), binding)
end

def create_member(identity, version)
  puts "Creating member %s version %s" % [identity, version]

  instance_home = member_dir(identity)

  FileUtils.mkdir_p(instance_home)

  gen_cert(identity)

  Dir.chdir(instance_home) do
    system("git clone -q file:///%s ." % COLLECTIVE_BASE)
    system("git checkout -q %s" % version)

    render_member_config(identity)
  end

  puts "Created a new instance %s in %s" % [identity, instance_home]
  puts "=" * 40
end

def member_dirs
  Dir.entries(COLLECTIVEDIR).reject do |dir|
    dir == "client.choria" || !dir.include?(".choria")
  end
end

def get_random_file(type)
  files = Dir.entries(File.join(BUILDER_TEMPLATES, type)).reject{|f| f.start_with?(".")}

  File.join(BUILDER_TEMPLATES, type, files[rand(files.size)])
end

def copy_classes
  member_dirs.each do |member|
    source = get_random_file("classes")
    puts "Copying classes from %s to %s" % [source, member]
    FileUtils.cp(source, File.join(member_dir(member), "etc", "classes.txt"))
  end
end

def copy_facts
  member_dirs.each do |member|
    source = get_random_file("facts")
    puts "Copying facts from %s to %s" % [source, member]
    FileUtils.cp(source, File.join(member_dir(member), "etc", "facts.yaml"))
  end
end

def member_dir(member)
  File.join(COLLECTIVEDIR, member)
end

def copy_plugins
  member_dirs.each do |member|
    puts "Copying plugins for instance %s" % member

    libdir = File.join(member_dir(member), "plugins")

    FileUtils.mkdir_p(libdir)
    FileUtils.cp_r(File.join(BUILDER_PLUGIN_SOURCE, "."), libdir)
  end

  puts "Copying plugins for client client.choria"
  FileUtils.cp_r(File.join(BUILDER_PLUGIN_SOURCE, "."), File.join(COLLECTIVE_CLIENT, "plugins"))
end

def member_pidfile(member)
  File.join(COLLECTIVE_PIDS, "%s.pid" % member)
end

def member_pid(member)
  pidfile = member_pidfile(member)

  return nil unless File.exist?(pidfile)

  Integer(File.read(pidfile).chomp)
end

def nats_pid_file(instance)
  File.join(COLLECTIVE_PIDS, "gnats-%d.pid" % instance)
end

def nats_pid(instance)
  pidfile = nats_pid_file(instance)

  return nil unless File.exist?(pidfile)

  Integer(File.read(pidfile).chomp)
end

def nats_running?(instance)
  return nil unless pid = nats_pid(instance)

  File.exist?("/proc/%s" % [pid])
end

def stop_nats
  3.times do |instance|
    pid_file = nats_pid_file(instance)

    puts "Stopping NATS on pid %d" % nats_pid(instance)

    unless nats_running?(instance)
      File.unlink(pid_file) if File.exist?(pid_file)
      return
    end

    Process.kill(2, nats_pid(instance))
    File.unlink(pid_file) if File.exist?(pid_file)
  end
end

def member_running?(member)
  pid = member_pid(member)

  return false unless pid

  # @todo check its mco
  File.exist?("/proc/%s" % pid)
end

def status
  puts "Collective Status:"
  puts

  3.times do |nats|
    if nats_running?(nats)
      puts "NATS instance %d: running pid %d" % [nats, nats_pid(nats)]
    else
      puts "NATS instance %d: stopped" % [nats]
    end
  end

  puts

  member_dirs.each do |member|
    if member_running?(member)
      puts "%s: running pid %d" % [member, member_pid(member)]
    else
      puts "%s: stopped" % [member]
    end
  end
end

def stop_member(member)
  return unless member_running?(member)

  pid = member_pid(member)
  pidfile = member_pidfile(member)

  return unless pid

  puts "Stopping collective member %s" % [member]

  Process.kill(2, pid)

  begin
    FileUtils.rm(pidfile) if File.exist?(pidfile)
  rescue
  end
end

def start_member(member)
  abort("No collective have been created, use 'rake create'") if member_dirs.empty?

  return true if member_running?(member)

  puts "Starting collective member %s" % [member]

  Dir.chdir(member_dir(member)) do
    system("MCOLLECTIVE_CERTNAME=%s ruby -I %s bin/mcollectived --config %s --pidfile %s" % [
      member,
      File.join(member_dir(member), "lib"),
      File.join(member_dir(member), "etc", "server.cfg"),
      File.join(COLLECTIVE_PIDS, "%s.pid" % member)
    ])
  end
end

def stop_all_members
  member_dirs.each do |member|
    stop_member(member)
  end
end

def start_all_members
  member_dirs.each do |member|
    start_member(member)
  end
end

def clean
  stop_all_members
  stop_nats

  FileUtils.rm_rf(Dir.glob("collective/*"))
end

desc "Sets up a subshell to use the new collective"
task :shell do
    abort("Don't know what shell to run you have no SHELL environment variable") unless ENV.include?("SHELL")
    abort("You need to create a collective first using rake create") if member_dirs.size == 0
    abort("NATS is not running, please start it first") unless nats_running?(0) && nats_running?(1) && nats_running?(2)

    client_home = member_dir("client.choria")
    client_config = File.join(client_home, "etc", "client.cfg")
    client_lib = File.join(client_home, "lib")
    client_bin = File.join(client_home, "bin")

    ENV["MCOLLECTIVE_EXTRA_OPTS"]= "--config=%s" % client_config
    ENV["RUBYLIB"] = client_lib
    ENV["MCOLLECTIVE_CERTNAME"] = "client.choria"
    ENV["CHORIA_SHELL"] = "1"

    puts
    puts "Running %s to start a subshell with MCOLLECTIVE_EXTRA_OPTS and RUBYLIB set" % ENV["SHELL"]
    puts
    puts "Please run the following once started: "
    puts
    puts "    PATH=%s:$PATH" % client_bin
    puts
    puts "To return to your normal shell and collective just type exit."
    puts

    system(ENV["SHELL"])
end

desc "Update the configuration and plugins of all collective members"
task :update do
  stop_all_members

  member_dirs.each do |member|
    render_member_config(member)
  end

  render_member_config("client.choria")

  copy_plugins
  copy_facts
  copy_classes
  start_all_members
end

desc "Restart all collective members"
task :restart do
  stop_all_members
  start_all_members
end

desc "Destroy the collective"
task :clean do
  clean
end

desc "Stop all members"
task :stop do
  stop_all_members
end

desc "Start all members"
task :start do
  start_all_members
end

desc "Show collective member statusses"
task :status do
  status
end

desc "Create a new Choria dev collective"
task :create do
  abort("No Plugin Source directory exist (%s)" % [BUILDER_PLUGIN_SOURCE]) unless File.exist?(BUILDER_PLUGIN_SOURCE)

  mc_repo        = ask("MCollective GIT Repository", "MC_SOURCE", "git://github.com/puppetlabs/marionette-collective.git")
  mc_branch      = ask("MCollective Branch Name", "MC_SOURCE_BRANCH", "2.9")
  mc_version     = ask("MCollective Version", "MC_VERSION", mc_branch == "master"? "master" : mc_branch)
  count          = Integer(ask("Instances To Create", "CHORIA_COUNT", 10))
  countstart     = Integer(ask("Instance Count Start", "CHORIA_COUNT_START", 0))
  prefix         = ask("Instance Name Prefix", "CHORIA_PREFIX", `hostname -s`.chomp)

  setup_base(mc_repo, mc_branch)

  count.times do |instance|
    create_member("%s-%d.choria" % [prefix, countstart + instance], mc_version)
  end

  create_member("client.choria", mc_version)

  copy_plugins
  copy_facts
  copy_classes

  puts
  puts "Created a collective with %s members:" % count
  puts
  puts "To recreate this collective use this command:"
  puts
  puts "  MC_SOURCE=%s \\" % mc_repo
  puts "  MC_SOURCE_BRANCH=%s \\" % mc_branch
  puts "  MC_VERSION=%s \\" % mc_version
  puts "  CHORIA_COUNT=%s \\" % count
  puts "  CHORIA_COUNT_START=%s \\" % countstart
  puts "  CHORIA_PREFIX=%s \\" % prefix
  puts "  rake create"
  puts
  puts "The collective instances and client are stored in collective/*"
  puts
  puts "Use rake start to start the collective, rake -T to see commands available to start,"
  puts "stop and update it."
  puts
end

desc "Sets up a 2 node NATS Cluster on localhost"
task :start_nats do
  `which gnatsd > /dev/null 2>&1`

  abort("Please ensure 'gnatsd' is in your PATH") unless $?.exitstatus == 0

  FileUtils.mkdir_p "logs"

  puts "Starting 3 NATS instances on localhost, use ^C to terminate"
  puts

  3.times do |index|
    abort("Already have a NATS instance %d" % index) if nats_running?(index)

    listen = 14222 + index
    monitor = 18222 + index
    cluster = 15222 + index

    instance_name = "nats-%d.choria" % index
    pid_file = File.join(COLLECTIVE_PIDS, "gnats-%d.pid" % index)
    log_file = "logs/nats-%d.log" % index

    puts "    TLS Certificate: %s" % cert_path(instance_name)
    puts "            TLS Key: %s" % private_key_path(instance_name)
    puts "     CA Certificate: %s" % ca_path
    puts "           PID File: %s" % pid_file
    puts "           Log File: %s" % log_file
    puts "        Listen Port: %d" % listen
    puts "       Monitor Port: %d" % monitor
    puts "       Cluster Port: %d" % cluster
    puts

    gen_cert(instance_name)

    system("nohup gnatsd -a localhost --tls --tlscert %s --tlskey %s --tlscacert %s --tlsverify -P %s -l %s -p %d -m %d --cluster nats://localhost:%d --routes nats://localhost:15222 -DV &" % [
      cert_path(instance_name), private_key_path(instance_name), ca_path, pid_file, log_file, listen, monitor, cluster
    ])
  end

  begin
    system("tail -qF logs/nats*log")
  rescue Interrupt
  ensure
    stop_nats
  end

end
