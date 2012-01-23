#! /usr/bin/env ruby

# This rakefile is meant to be run from within the [Puppet Win
# Builder](http://links.puppetlabs.com/puppetwinbuilder) tree.

# Load Rake
begin
  require 'rake'
rescue LoadError
  require 'rubygems'
  require 'rake'
end

require 'rake/clean'

# Add extensions to core Ruby classes
require 'rake/core_extensions'

# Added download task from buildr
require 'rake/downloadtask'
require 'rake/extracttask'
require 'rake/checkpoint'
require 'rake/env'

# Ensure '.' is in the LOAD_PATH in Ruby 1.9.2
$LOAD_PATH.unshift File.expand_path(File.dirname(__FILE__))

# Where we're situated in the filesystem relative to the Rakefile
TOPDIR=File.dirname(__FILE__)

CLOBBER.include('tmp/*')
CLEAN.include('stagedir/*')
CLEAN.include('wix/fragments/*.wxs')
CLEAN.include('wix/**/*.wixobj')
CLEAN.include('pkg/*')

# These are file tasks that behave like mkdir -p
directory 'pkg'
directory 'tmp'
directory 'stagedir/sys'
directory 'wix/fragments'

# Produce a wixobj from a wxs file.
def candle(wxs_file, basedir)
  Dir.chdir File.join(TOPDIR, File.dirname(wxs_file)) do
    sh "candle -dStageDir=#{basedir} #{File.basename(wxs_file)}"
  end
end

# Produce a wxs file from a directory in the stagedir
# e.g. heat('wxs/fragments/foo.wxs', 'stagedir/sys/foo')
def heat(wxs_file, stage_dir)
  Dir.chdir TOPDIR do
    cg_name = File.basename(wxs_file.ext(''))
    dir_ref = File.basename(File.dirname(stage_dir))
    sh "heat dir #{stage_dir} -v -ke -indent 2 -cg #{cg_name} -gg -dr #{dir_ref} -var var.StageDir -out #{wxs_file}"
  end
end

def unzip(zip_file, dir)
  Dir.chdir TOPDIR do
    Dir.chdir dir do
      sh "7za -y x #{File.join(TOPDIR, zip_file)}"
    end
  end
end

## File Lists

TARGETS = FileList['pkg/puppet.msi']

# These translate to ZIP files we'll download
# FEATURES = %w{ ruby git wix misc }
FEATURES = %w{ ruby }
DOWNLOADS = FEATURES.collect { |fn| File.join("tmp", fn.ext('zip')) }

# There is a 1:1 mapping between a wxs file and a wixobj file

# These files should be committed to VCS
WXSFILES = FileList['wix/*.wxs']
# These files should be auto-generated by heat
WXSFRAGMENTS = FEATURES.collect { |fn| File.join("wix", "fragments", fn.ext('wxs')) }
# All of the objects we need to create
WIXOBJS = (WXSFILES + WXSFRAGMENTS).ext('wixobj')

# These directories should be unpacked into stagedir/sys
SYSTOOLS = FEATURES.collect { |fn| File.join("stagedir", "sys", fn) }

task :default => :build
# High Level Tasks.  Other tasks will add themselves to these tasks
# dependencies.

# This is also called from the build script in the Puppet Win Builder archive.
# This will be called AFTER the update task in a new process.
desc "Build puppet.msi"
task :build => "pkg/puppet.msi"

desc "Download example"
task :download => DOWNLOADS

desc "Unzip and stage sys tools"
task :unzip => SYSTOOLS

desc "List available rake tasks"
task :help do
  sh 'rake -T'
end

# The update task is always called from the build script
# This gives the repository an opportunity to update itself
# and manage how it updates itself.
desc "Update the build scripts"
task :update do
  sh 'git pull'
end

# Tasks to unpack the zip files
SYSTOOLS.each do |systool|
  zip_file = File.join("tmp", File.basename(systool).ext('zip'))
  file systool => [ zip_file, File.dirname(systool) ] do
    unzip(zip_file, File.dirname(systool))
  end
end

DOWNLOADS.each do |fn|
  file fn => [ File.dirname(fn) ] do |t|
    download t.name => "http://downloads.puppetlabs.com/development/ftw/#{File.basename(t.name)}"
  end
end

WIXOBJS.each do |wixobj|
  stagedir = File.join(TOPDIR, 'stagedir', 'sys')
  file wixobj => [ wixobj.ext('wxs'), File.dirname(wixobj) ] do |t|
    candle t.name.ext('wxs'), File.join(stagedir, File.basename(t.name.ext(''))).gsub('/', '\\')
  end
end

WXSFRAGMENTS.each do |wxs_frag|
  source_dir = File.join('stagedir', 'sys', File.basename(wxs_frag).ext(''))
  file wxs_frag => [ source_dir, File.dirname(wxs_frag) ] do |t|
    heat t.name, source_dir
  end
end

####### REVISIT

file 'pkg/puppet.msi' => WIXOBJS do |t|
  sh "light #{t.prerequisites.join(' ')} -out #{t.name}"
end

