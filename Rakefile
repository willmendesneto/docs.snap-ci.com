#!/usr/bin/env rake -f

require 'yaml'


def approx_version_string(str)
  Gem::Version.new(str).segments.first(3).join('.')
end

desc "detect versions"
task :detect_versions do

  unless File.exist?('/etc/centos-release')
    $stderr.puts("Not performing any version detection because we are not running on centos.")
    next
  end

  config = YAML.load_file('_config.yml')

  versions = {
    'arch'   => %x[uname -m].strip,
    'centos' => File.read('/etc/centos-release').match(/release ([\d\.]+)/)[1],
    'kernel' => %x[uname -r].strip,
    'ant'    => %x[ant -version].match(/version (.*) compiled/)[1],
    'maven'  => %x[mvn --version].match(/Apache Maven (.*) \(r/)[1],
    'gradle' => %x[gradle --version].match(/^Gradle (.*)$/)[1]
  }

  development_tools = %w(git gcc gcc-c++ make)
  development_libs  = %w(openssl libxml2 libxslt ImageMagick qt48-qt)
  sql_sdatabases    = %w(mysql postgresql91 sqlite)
  third_party_tools = %w(heroku-toolbelt)
  no_sql_databases  = %w(couchdb redis mongo-10gen)
  languages         = %w(nodejs)
  browser_tools     = %w(phantomjs google-chrome-stable firefox)

  rpms = development_tools + development_libs + sql_sdatabases + no_sql_databases + languages + third_party_tools + browser_tools

  gems = %w(rake bundler)

  rpms.each do |pkg|
    versions[pkg] = approx_version_string(%x[rpm -q --queryformat '%{VERSION}' #{pkg}])
    # raise "Could not detect version of package #{pkg}" unless $?.success?
  end

  versions['ruby'] = %x[rpm -q --queryformat '%{NAME} ' $(rpm -qa | egrep 'ruby-[0-9]+' | sort)].strip.gsub('ruby-', '').split

  versions['openjdk'] = Dir['/usr/lib/jvm/java-*-openjdk.x86_64/bin/java'].collect do |java|
    version = %x[#{java} -version 2>&1].match(/java version "(.*)"/)[1]
  end

  versions['sunjdk'] = Dir['/usr/lib/jvm/java-*-openjdk.x86_64/bin/java'].collect do |java|
    version = %x[#{java} -version 2>&1].match(/java version "(.*)"/)[1]
  end

  gems.each do |gem|
    mkdir_p 'tmp'
    gem_spec = %x[unset GEM_HOME GEM_PATH BUNDLE_GEMFILE BUNDLE_BIN_PATH RUBYOPT; gem specification --ruby #{gem} > tmp/#{gem}.gemspec]
    spec = Gem::Specification.load("tmp/#{gem}.gemspec")
    versions[gem] = spec.version.to_s
  end

  config['versions'] = versions

  File.open("_config_with_version.yaml", 'w') {|f| f.puts config.to_yaml}

  # ant maven gradle
end

desc "build all documentation"
task :build => :detect_versions do
  sh("bundle exec jekyll build --config _config_with_version.yaml")
end

desc "deploy the documentation"
task :deploy do
  sh("curl --location http://sourceforge.net/projects/s3tools/files/s3cmd/1.0.1/s3cmd-1.0.1.tar.gz 2>/dev/null | tar -zxf -")
  File.open('s3.cfg', 'w') do |f|
    f.puts "[default]"
    f.puts "access_key = #{ENV['S3_ACCESS_KEY']}"
    f.puts "secret_key = #{ENV['S3_SECRET_KEY']}"
  end

  sh("./s3cmd-1.0.1/s3cmd sync --config s3.cfg --verbose --acl-public --delete-removed --no-preserve _site/* s3://#{ENV['S3_BUCKET']}")
end