require 'rubygems'
require 'rake/testtask'
require 'rspec/core/rake_task'
require 'rubygems/package_task' 
require 'rdoc/task' 

load 'dry_crud.gemspec'

TEST_APP_ROOT  = File.join(File.dirname(__FILE__), 'test', 'test_app')
GENERATOR_ROOT = File.join(File.dirname(__FILE__), 'lib', 'generators', 'dry_crud')

task :default => :test

desc "Run all tests"
task :test => ['test:unit', 'test:spec']


namespace :test do

  desc "Run Test::Unit tests"
  Rake::TestTask.new(:unit => 'test:app:init') do |test| 
    test.libs << "test/test_app/test" 
    test.test_files = Dir[ "test/test_app/test/**/*_test.rb" ] 
    test.verbose = true
  end
  
  desc "Run RSpec tests"
  RSpec::Core::RakeTask.new(:spec => 'test:app:init') do |t|
    t.ruby_opts = "-I test/test_app/spec"
    t.pattern = "test/test_app/spec/**/*_spec.rb"
  end


  namespace :app do
    task :environment do
      ENV['RAILS_ROOT'] = TEST_APP_ROOT
      ENV['RAILS_ENV'] = 'test'
      
      require(File.join(TEST_APP_ROOT, 'config', 'environment'))
    end
  
    desc "Create a rails test application"
    task :create do
      unless File.exist?(TEST_APP_ROOT)
        sh "rails new #{TEST_APP_ROOT} --skip-bundle"
        FileUtils.cp(File.join(File.dirname(__FILE__), 'test', 'templates', 'Gemfile'), TEST_APP_ROOT)
        sh "cd #{TEST_APP_ROOT}; bundle install --local" # update Gemfile.lock
        sh "cd #{TEST_APP_ROOT}; rails g rspec:install"
        FileUtils.rm_f(File.join(TEST_APP_ROOT, 'test', 'performance', 'browsing_test.rb'))
      end
    end
      
    desc "Run the dry_crud generator for the test application"
    task :generate_crud => [:create, :environment] do
      require File.join(GENERATOR_ROOT, 'dry_crud_generator')
    
      DryCrudGenerator.new([], {:force => true, 
                                :templates => ENV['HAML'] ? 'haml' : 'erb', 
                                :tests => 'all'},
                               :destination_root => TEST_APP_ROOT).invoke_all
    end
   
    desc "Populates the test application with some models and controllers"
    task :populate => :generate_crud do
      # copy test app templates
      FileUtils.cp_r(File.join(File.dirname(__FILE__), 'test', 'templates', '.'), TEST_APP_ROOT)
      
      # copy shared fixtures
      FileUtils.cp_r(File.join(File.dirname(__FILE__), 'test', 'templates', 'test', 'fixtures'),
                     File.join(TEST_APP_ROOT, 'spec'))
      
      # replace some unused files
      FileUtils.rm_f(File.join(TEST_APP_ROOT, 'public', 'index.html'))
      layouts = File.join(TEST_APP_ROOT, 'app', 'views', 'layouts')
      FileUtils.mv(File.join(layouts, 'crud.html.erb'),
                   File.join(layouts, 'application.html.erb'), 
                   :force => true) if File.exists?(File.join(layouts, 'crud.html.erb'))
      FileUtils.mv(File.join(layouts, 'crud.html.haml'),
                   File.join(layouts, 'application.html.haml'), 
                   :force => true) if File.exists?(File.join(layouts, 'crud.html.haml'))
                   
      # remove unused template type, erb or haml
      exclude = ENV['HAML'] ? 'erb' : 'haml'
      Dir.glob(File.join(TEST_APP_ROOT, 'app', 'views', '**', "*.#{exclude}")).each do |f|
        FileUtils.rm(f)
      end
    end
    
    desc "Insert seed data into the test database"
    task :seed => :populate do
      # migrate the database
      FileUtils.cd(TEST_APP_ROOT) do
        sh "rake db:migrate db:seed RAILS_ENV=development --trace"
        sh "rake db:migrate RAILS_ENV=test --trace"  # db:test:prepare does not work for jdbcsqlite3
      end
    end
   
    desc "Initializes the test application with a couple of classes"
    task :init => [:seed, 
                   :customize]
                   
    desc "Customize some of the functionality provided by dry_crud"
    task :customize => ['test:app:add_pagination'
                        #'test:app:use_bootstrap'
                       ]
    
    desc "Adds pagination to the test app"
    task :add_pagination => :generate_crud do
      list_ctrl = File.join(TEST_APP_ROOT, 'app', 'controllers', 'list_controller.rb')
      file_replace(list_ctrl, /def list_entries\n\s+model_scope\s*\n/, 
                              "def list_entries\n    model_scope.page(params[:page]).per(10)\n")
      file_replace(File.join(TEST_APP_ROOT, 'app', 'views', 'list', 'index.html.erb'), 
                   "<%= render 'list' %>", "<%= paginate entries %>\n\n<%= render 'list' %>")
      file_replace(File.join(TEST_APP_ROOT, 'app', 'views', 'list', 'index.html.haml'), 
                   "= render 'list'", "= paginate entries\n\n= render 'list'")
    end
    
    desc "Use Boostrap in the test app"
    task :use_bootstrap => :generate_crud do
       file_replace(File.join(TEST_APP_ROOT, 'app', 'assets', 'stylesheets', 'application.css'), " *= require_self", "*= require twitter/bootstrap\n *= require_self")
       file_replace(File.join(TEST_APP_ROOT, 'app', 'assets', 'javascripts', 'application.js'), "//= require_tree .", "//= require twitter/bootstrap\n//= require_tree .")
       FileUtils.rm(File.join(TEST_APP_ROOT, 'app', 'assets', 'stylesheets', 'sample.scss'))
       
       layouts = File.join(TEST_APP_ROOT, 'app', 'views', 'layouts')
       FileUtils.mv(File.join(layouts, 'bootstrap.html.erb'),
                    File.join(layouts, 'application.html.erb'), 
                    :force => true) if File.exists?(File.join(layouts, 'bootstrap.html.erb'))
       FileUtils.mv(File.join(layouts, 'bootstrap.html.haml'),
                    File.join(layouts, 'application.html.haml'), 
                    :force => true) if File.exists?(File.join(layouts, 'bootstrap.html.haml'))
    end
   
  end
end

desc "Clean up all generated resources"
task :clobber do
  FileUtils.rm_rf(TEST_APP_ROOT)
end

desc "Install dry_crud as a local gem." 
task :install => [:package] do
  sudo = RUBY_PLATFORM =~ /win32/ ? '' : 'sudo' 
  gem = RUBY_PLATFORM =~ /java/ ? 'jgem' : 'gem' 
  sh %{#{sudo} #{gem} install --no-ri pkg/dry_crud-#{File.read('VERSION').strip}.gem}
end

desc "Deploy rdoc to website"
task :site => :rdoc do
  if ENV['DEST']
    sh "rsync -rzv rdoc/ #{ENV['DEST']}"
  else
    puts "Please specify a destination with DEST=user@server:/deploy/dir"
  end
end

if RUBY_VERSION == '1.8.7' && RUBY_PLATFORM != 'java'
	require 'rcov/rcovtask'
	Rcov::RcovTask.new do |test| 
	    test.libs << "test/test_app/test"
	    test.test_files = Dir[ "test/test_app/test/**/*_test.rb" ]
	    test.rcov_opts = ['--text-report',
	                      '-i', '"test\/test_app\/app\/.*"',
	                      '-x', '"\/Library\/Ruby\/.*"']
	    test.verbose = true
	end
end

# :package task
Gem::PackageTask.new(DRY_CRUD_GEMSPEC) do |pkg|
  if Rake.application.top_level_tasks.include?('release')
    pkg.need_tar_gz = true 
    pkg.need_tar_bz2 = true 
    pkg.need_zip  = true
  end 
end

# :rdoc task
Rake::RDocTask.new do |rdoc| 
  rdoc.title  = 'Dry Crud' 
  rdoc.options << '--line-numbers'
  rdoc.rdoc_files.include(*FileList.new('*') do |list|
    list.exclude(/(^|[^.a-z])[a-z]+/)
    list.exclude('TODO') 
    end.to_a)
  rdoc.rdoc_files.include('lib/generators/dry_crud/templates/**/*.rb') 
  rdoc.rdoc_files.exclude('lib/generators/dry_crud/templates/**/*_test.rb') 
  rdoc.rdoc_files.exclude('TODO') 
    
  rdoc.rdoc_dir = 'rdoc'
  rdoc.main = 'README.rdoc' 
end

desc "Outputs the commands required for a release. Does not perform any other actions"
task :release do
  version = File.read('VERSION').strip
  puts "Issue the following commands to perform a release:"
  puts " $ git tag version-#{version} -m \"Version tag for dry_crud-#{version}.gem\""
  puts " $ git push --tags"
  puts " $ rake repackage"
  puts " $ gem push pkg/dry_crud-#{version}.gem"
end


def file_replace(file, expression, replacement)
  return unless File.exists?(file)
  text = File.read(file)
  replaced = text.gsub(expression, replacement)
  puts "WARN: Nothing replaced in '#{file}' for '#{expression}'" if text == replaced
  File.open(file, "w") { |f| f.puts replaced }
end
