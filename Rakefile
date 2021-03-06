require 'cgi'
require 'json'
require 'net/http'

# TODO:  private boxes may need to specify a mirror

# Enables `bundle exec rake do_all[ubuntu-12.04-amd64,centos-7.1-x86_64]
# http://blog.stevenocchipinti.com/2013/10/18/rake-task-with-an-arbitrary-number-of-arguments/
task :do_all do |task, args|
  args.extras.each do |a|
    # build stage
    Rake::Task['build_box'].invoke(a)
    Rake::Task['build_box'].reenable
  end

  # publish stage
  Rake::Task['upload_all'].invoke
  Rake::Task['upload_all'].reenable

  # release stage
  Rake::Task['release_all'].invoke
  Rake::Task['release_all'].reenable

  # revoke
  Rake::Task['revoke_all'].invoke
  Rake::Task['revoke_all'].reenable

  # delete
  Rake::Task['delete_all'].invoke
  Rake::Task['delete_all'].reenable
end

# bundle exec rake build_box[ubuntu-12.04-amd64]
desc 'Build a single bento box'
task :build_box, :template do |t, args|
  sh "#{build_command(args[:template])}"
end

desc 'Upload boxes'
task :upload_all do
  metadata_files.each do |metadata_file|
    puts "Processing #{metadata_file} for upload."
    Rake::Task['upload_box'].invoke(metadata_file)
    Rake::Task['upload_box'].reenable
  end
end

desc 'Upload box files for all providers'
task :upload_box, :metadata_file do |f, args|
  metadata = box_metadata(args[:metadata_file])
  create_box(metadata['name'])
  create_box_version(metadata['name'], metadata['version'])
  create_providers(metadata['name'], metadata['version'], metadata['providers'].keys)
  upload_to_atlas(metadata['name'], metadata['version'], metadata['providers'])
end

desc 'Release all boxes for a version'
task :release_all do
  metadata_files.each do |metadata_file|
    puts "Processing #{metadata_file} for release."
    metadata = box_metadata(metadata_file)
    release_version(metadata['name'], metadata['version'])
  end
end

desc 'Revoke all boxes for a version'
task :revoke_all, :version do |v, args|
  revoke_version(args[:version])
end

desc 'Delete all boxes for a version'
task :delete_all, :version do |v, args|
  delete_version(args[:version])
end
desc 'Clean the build directory'
task :clean do
  `rm -rf builds/*.{box,json}`
end

def atlas_api
  @atlas_api ||= 'https://atlas.hashicorp.com/api/v1'
end

def atlas_org
  @atlas_org ||= ENV['ATLAS_ORG']
end

def atlas_token
  @atlas_token ||= ENV['ATLAS_TOKEN']
end

def class_for_request(verb)
  Net::HTTP.const_get(verb.to_s.capitalize)
end
def build_uri(verb, path, params = {})
  if %w(delete, get).include?(verb)
    path = [path, to_query_string(params)].compact.join('?')
  end

  # Parse the URI
  uri = URI.parse(path)

  # Don't merge absolute URLs
  uri = URI.parse(File.join(endpoint, path)) unless uri.absolute?

  # Return the URI object
  uri
end

def to_query_string(hash)
  hash.map do |key, value|
    "#{CGI.escape(key)}=#{CGI.escape(value)}"
  end.join('&')[/.+/]
end

def request(verb, url, data = {}, headers = {})
  uri = build_uri(verb, url, data)

  # Build the request.
  request = class_for_request(verb).new(uri.request_uri)
  if %w(patch post put delete).include?(verb)
    if data.respond_to?(:read)
      request.content_length = data.size
      request.body_stream = data
    elsif data.is_a?(Hash)
      request.form_data = data
    else
      request.body = data
    end
  end

  # Add headers
  headers.each do |key, value|
    request.add_field(key, value)
  end

  connection = Net::HTTP.new(uri.host, uri.port)

  if uri.scheme == 'https'
    require 'net/https' unless defined?(Net::HTTPS)

    # Turn on SSL
    connection.use_ssl = true
    connection.verify_mode = OpenSSL::SSL::VERIFY_PEER
  end

  connection.start do |http|
    response = http.request(request)

    case response
      when Net::HTTPRedirection
        redirect = URI.parse(response['location'])
        request(verb, redirect, data, headers)
      else
        response
      end
  end
end

def box_metadata(metadata_file)
  metadata = Hash.new
  file = File.read(metadata_file)
  json = JSON.parse(file)

  # metadata needed for upload:  boxname, version, provider, box filename
  metadata['name'] = json['name']
  metadata['version'] = json['version']
  metadata['box_basename'] = json['box_basename']
  metadata['providers'] = Hash.new
  json['providers'].each do |provider|
    metadata['providers'][provider['name']] = provider.reject { |k, _| k == 'name' }
  end
  metadata
end

def build_command(template)
  cmd = %[./bin/bento build]
  providers = %W[vmware-iso parallels-iso virtualbox-iso]
  providers.delete(vmware-iso) if which('vmrun')
  providers.delete(parallels-iso) if which('prlctl')
  cmd.insert(1, "--except #{providers.join(",")}")
  cmd.insert(1, "--mirror #{ENV['PACKER_MIRROR']}") if private?(template)
  cmd.insert(1, "#{template}.json")
  cmd
end

def metadata_files
  @metadata_files ||= compute_metadata_files
end

def compute_metadata_files
  `ls builds/*.json`.split("\n")
end

def create_box(boxname)
  req = request('get', "#{atlas_api}/box/#{atlas_org}/#{boxname}", { 'box[username]' => atlas_org, 'access_token' => atlas_token } )
  if req.code.eql?('404')
    if private?(boxname)
      puts "Creating the private box #{boxname} in atlas."
      req = request('post', "#{atlas_api}/boxes", { 'box[name]' => boxname, 'box[username]' => atlas_org, 'access_token' => atlas_token, 'is_private' => true }, { 'Content-Type' => 'application/json' } )
    else
      puts "Creating the public box #{boxname} in atlas."
      req = request('post', "#{atlas_api}/boxes", { 'box[name]' => boxname, 'box[username]' => atlas_org, 'access_token' => atlas_token }, { 'Content-Type' => 'application/json' } )
    end
  else
    puts "The box #{boxname} exists in atlas, continuing."
  end
end

def create_box_version(boxname, version)
  req = request('post', "#{atlas_api}/box/#{atlas_org}/#{boxname}/versions", { 'version[version]' => version, 'access_token' => atlas_token },{ 'Content-Type' => 'application/json' } )

  puts "Created box version #{boxname} #{version}." if req.code == '200'
  puts "Box version #{boxname} #{version} already exists, continuing." if req.code == '422'
end

def create_providers(boxname, version, provider_names)
  provider_names.each do |provider|
    puts "Creating provider #{provider} for #{boxname} #{version}"
    req = request('post', "#{atlas_api}/box/#{atlas_org}/#{boxname}/version/#{version}/providers", { 'provider[name]' => provider, 'access_token' => atlas_token }, { 'Content-Type' => 'application/json' }  )
    puts "Created #{provider} for #{boxname} #{version}" if req.code == '200'
    puts "Provider #{provider} for #{boxname} #{version} already exists, continuing." if req.code == '422'
  end
end

def upload_to_atlas(boxname, version, providers)
  # Extract the upload path
  providers.each do |provider, provider_data|
    boxfile = provider_data['file']
    # Get the upload path.
    req = request('get', "#{atlas_api}/box/#{atlas_org}/#{boxname}/version/#{version}/provider/#{provider}/upload?access_token=#{atlas_token}")
    upload_path = JSON.parse(req.body)['upload_path']
    token = JSON.parse(req.body)['token']

    # Upload the box.
    puts "Uploading the box #{boxfile} to atlas box: #{boxname}, version: #{version}, provider: #{provider}, upload path: #{upload_path}"
    upload_request = request('put', upload_path, File.open("builds/#{boxfile}"))

    # Verify the download token
    req = request('get', "#{atlas_api}/box/#{atlas_org}/#{boxname}/version/#{version}/provider/#{provider}?access_token=#{atlas_token}")
    hosted_token = JSON.parse(req.body)['hosted_token']

    if token == hosted_token
      puts "Successful upload of box #{boxfile} to atlas box: #{boxname}, version: #{version}, provider: #{provider}"
    else
      puts "Failed upload due to non-matching tokens of box #{boxfile} to atlas box: #{boxname}, version: #{version}, provider: #{provider}"
      # need to fail the rake task
    end
 end
end

def release_version(boxname, version)
  puts "Releasing version #{version} of box #{boxname}"
  req = request('put', "#{atlas_api}/box/#{atlas_org}/#{boxname}/version/#{version}/release", { 'access_token' => atlas_token }, { 'Content-Type' => 'application/json' })
  puts "Version #{version} of box #{boxname} has been successfully released" if req.code == '200'
end

def revoke_version(version)
  org = request('get', "#{atlas_api}/user/#{atlas_org}?access_token=#{atlas_token}")
  boxes = JSON.parse(org.body)['boxes']

  boxes.each do |b|
    b['versions'].each do |v|
      if v['version'] == version
        puts "Revoking version #{v['version']} of box #{b['name']}"
        req = request('put', v['revoke_url'], { 'access_token' => atlas_token }, { 'Content-Type' => 'application/json' })
        if req.code == '200'
          puts "Version #{v['version']} of box #{b['name']} has been successfully revoked" 
        else
          puts "Something went wrong #{req.code}"
        end 
      end
    end
  end
end

def delete_version(version)
  org = request('get', "#{atlas_api}/user/#{atlas_org}?access_token=#{atlas_token}")
  boxes = JSON.parse(org.body)['boxes']

  boxes.each do |b|
    b['versions'].each do |v|
      if v['version'] == version
        puts "Deleting version #{v['version']} of box #{b['name']}"
        puts "#{atlas_api}/box/#{atlas_org}/#{b['name']}/version/#{v['version']}"
        req = request('delete', "#{atlas_api}/box/#{atlas_org}/#{b['name']}/version/#{v['version']}", { 'access_token' => atlas_token }, { 'Content-Type' => 'application/json' })
        if req.code == '200'
          puts "Version #{v['version']} of box #{b['name']} has been successfully deleted" 
        else
          puts "Something went wrong #{req.code} #{req.body}"
        end 
      end
    end
  end
end

# http://stackoverflow.com/questions/2108727/which-in-ruby-checking-if-program-exists-in-path-from-ruby
def which(cmd)
  exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
  ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
    exts.each { |ext|
      exe = File.join(path, "#{cmd}#{ext}")
      return exe if File.executable?(exe) && !File.directory?(exe)
    }
  end
  return false
end

#
# private boxes
#
def private?(boxname)
  proprietary_os_list = %w(macosx sles solaris windows)
  proprietary_os_list.any? { |p| boxname.include?(p) }
end
