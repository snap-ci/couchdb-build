require 'rubygems'
require 'bundler/setup'

require 'rake/clean'

distro = nil
fpm_opts = ""

if File.exist?('/etc/system-release') && File.read('/etc/redhat-release') =~ /centos|redhat|fedora|amazon/i
  distro = 'rpm'
  fpm_opts << " --rpm-user couchdb --rpm-group couchdb "
elsif File.exist?('/etc/os-release') && File.read('/etc/os-release') =~ /ubuntu|debian/i
  distro = 'deb'
  fpm_opts << " --deb-user couchdb --deb-group couchdb "
end

unless distro
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

version = "1.6.1"
release = Time.now.utc.strftime('%Y%m%d%H%M%S')
name = "couchdb-#{version}"
before_install = File.expand_path("../hooks/before-install", __FILE__)

description_string = %Q{CouchDB is an open source database that focuses on ease of use and on being a database that completely embraces the web. It is a document-oriented NoSQL database that uses JSON to store data, JavaScript as its query language using MapReduce, and HTTP for an API.}

jailed_root = File.expand_path('../jailed-root', __FILE__)

CLEAN.include("downloads")
CLEAN.include("jailed-root")
CLEAN.include("log")
CLEAN.include("pkg")

task :init do
  mkdir_p "log"
  mkdir_p "pkg"
  mkdir_p "downloads"
  mkdir_p "jailed-root"
end

task :download do
  cd 'downloads' do
    sh("git clone --quiet https://github.com/apache/couchdb couchdb-#{version}")
    cd ("couchdb-#{version}") do
      sh("git checkout tags/#{version}")
    end
  end
end

task :dependencies do
  if distro == "deb"
    sh("sudo apt-get -y update")
    sh("sudo apt-get -y install libicu-dev pkg-config libmozjs185-dev help2man")
    sh("sudo apt-get -y install libtool automake autoconf autoconf-archive")
    sh("sudo apt-get -y install texlive-latex-base texlive-latex-recommended")
    sh("sudo apt-get -y install texlive-latex-extra texlive-fonts-recommended texinfo")
    sh("sudo apt-get -y install python-pygments python-docutils python-sphinx")

    sh("sudo apt-get install -y erlang-base-hipe")
    sh("sudo apt-get install -y erlang-dev")
    sh("sudo apt-get install -y erlang-manpages")
    sh("sudo apt-get install -y erlang-eunit")
    sh("sudo apt-get install -y erlang-nox")
  else
    sh("sudo yum install -y autoconf")
    sh("sudo yum install -y autoconf-archive")
    sh("sudo yum install -y automake")
    sh("sudo yum install -y curl-devel")
    sh("sudo yum install -y erlang-asn1")
    sh("sudo yum install -y erlang-erts")
    sh("sudo yum install -y erlang-eunit")
    sh("sudo yum install -y erlang-os_mon")
    sh("sudo yum install -y erlang-xmerl")
    sh("sudo yum install -y help2man")
    sh("sudo yum install -y js-devel")
    sh("sudo yum install -y libicu-devel")
    sh("sudo yum install -y libtool")
    sh("sudo yum install -y perl-Test-Harness")
  end
end

task :configure do
  cd "downloads" do
    cd "couchdb-#{version}" do
      sh("./bootstrap")
      sh "./configure --prefix=/opt/local/couchdb/#{version} > #{File.dirname(__FILE__)}/log/configure.#{version}.log 2>&1"
    end
  end
end

task :make do
  num_processors = %x[nproc].chomp.to_i
  num_jobs       = num_processors + 1

  cd "downloads/couchdb-#{version}" do
    sh("make -j#{num_jobs} > #{File.dirname(__FILE__)}/log/make.#{version}.log 2>&1")
  end
end

task :make_install do
  rm_rf  jailed_root
  mkdir_p jailed_root
  cd "downloads/couchdb-#{version}" do
    sh("sudo make install DESTDIR=#{jailed_root} > #{File.dirname(__FILE__)}/log/make-install.#{version}.log 2>&1")
  end
end


task :dist do
  require 'erb'
  class RpmSpec
    attr_accessor :version, :release
    def initialize(version, release)
      @version      = version
      @release      = release
    end

    def get_binding
      binding
    end
  end

  mkdir_p "#{jailed_root}/usr/local/bin"
  mkdir_p "#{jailed_root}/etc/default"
  mkdir_p "#{jailed_root}/etc/init.d"

  cd "#{jailed_root}/usr/local/bin" do
    Dir["../../../opt/local/couchdb/#{version}/bin/*"].each do |bin_file|
      ln_sf bin_file, File.basename(bin_file)
    end
  end

  cd "#{jailed_root}/etc/init.d" do
    Dir["../../opt/local/couchdb/#{version}/etc/init.d/*"].each do |bin_file|
      ln_sf bin_file, File.basename(bin_file)
    end
  end

  cd "#{jailed_root}/etc/default" do
    Dir["../../opt/local/couchdb/#{version}/etc/default/*"].each do |bin_file|
      ln_sf bin_file, File.basename(bin_file)
    end
  end



  cd "pkg" do
    sh(%Q{
      bundle exec fpm -s dir -t #{distro} --name couchdb-#{version} -a x86_64 --version "#{version}" -C #{jailed_root} --verbose #{fpm_opts} --maintainer snap-ci@thoughtworks.com --vendor snap-ci@thoughtworks.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license 'GPLv2' --before-install #{before_install} .
    })
  end
end

desc "build couchdb rpm/deb"
task :default => [:clean, :init, :download, :dependencies, :configure, :make, :make_install, :dist]
