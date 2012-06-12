%w{ bundler/setup rubygems fileutils uri net/http tmpdir digest/md5 ./doc/markit }.each do |file|
	require file
end

VERSION = '0.5-SNAPSHOT'
BASE_DIR = File.expand_path('.', File.dirname(__FILE__)) 
TARGET_DIR 	= "#{BASE_DIR}/target" 
BUILD_DIR 	= "#{BASE_DIR}/target/build"
CACHE_DIR 	= "#{BASE_DIR}/target/cache"
ZIP_EXE = 'C:\Program Files\7-Zip\7z.exe'
	

desc 'cleans all output and cache directories'
task :clean do 
	FileUtils.rm_rf TARGET_DIR
end

desc 'downloads required resources and builds the devpack binary'
task :build do
	recreate_dirs
	download_tools
	# download_boxes
	download_installables
	copy_files
	generate_docs
	install_gems
	clone_repositories
	# assemble_kitchen
end


def recreate_dirs
	FileUtils.rm_rf BUILD_DIR
	%w{ boxes docs home install repo tools }.each do |dir|
		FileUtils.mkdir_p "#{BUILD_DIR}/#{dir}"
	end
	FileUtils.mkdir_p CACHE_DIR
end

def copy_files
	FileUtils.cp_r Dir.glob("#{BASE_DIR}/files/*"), "#{BUILD_DIR}"
end

def generate_docs
	Dir.glob("#{BASE_DIR}/*.md").each do |md_file|
		html = MarkIt.to_html(IO.read(md_file))
		outfile = "#{BUILD_DIR}/_#{File.basename(md_file, '.md')}.html"
		File.open(outfile, 'w') {|f| f.write(html) }
	end
end

def download_tools
	[
		%w{ cloud.github.com/downloads/adoxa/ansicon/ansi151.zip 										ansicon },
		%w{ dfn.dl.sourceforge.net/project/console/console-devel/2.00/Console-2.00b148-Beta_32bit.zip 	console2 },
		%w{ www.holistech.co.uk/sw/hostsedit/hostsedit.zip 												hostedit },
		%w{ c758482.r82.cf2.rackcdn.com/Sublime%20Text%202%20Build%202181%20x64.zip 					sublimetext2 },
		%w{ msysgit.googlecode.com/files/PortableGit-1.7.10-preview20120409.7z							portablegit },
		%w{ files.vagrantup.com/packages/eb590aa3d936ac71cbf9c64cf207f148ddfc000a/vagrant_1.0.3.msi 	vagrant },
		%w{ miked.ict.rave.ac.uk/download/attachments/589834/OpenSSH_for_Windows_5.6p1-2.exe 			sshwindows 
			ssh.exe|scp.exe|cygz.dll|cygwin1.dll|cygssp-0.dll|cyggcc_s-1.dll|cygcrypto-0.9.8.dll },
		%w{ switch.dl.sourceforge.net/project/kdiff3/kdiff3/0.9.96/KDiff3Setup_0.9.96.exe 				kdiff3 
			kdiff3.exe }
	]
	.each do |host_and_path, target_dir, includes = ''|
		download_and_unpack "http://#{host_and_path}", "#{BUILD_DIR}/tools/#{target_dir}", includes.split('|')		
	end
end

def download_boxes
	%w{ 
		ubuntu-12.04-server-amd64-bare-os.box 
		ubuntu-12.04-server-amd64-vagrant.box
		chef-server-on-ubuntu-12.04-server-amd64-vagrant
	}
	.each do |file|
		download "http://dl.dropbox.com/u/13494216/#{file}.box", "#{BUILD_DIR}/boxes/#{file}.box"		
	end
end

def download_installables
	%w{ 
		www.gringod.com/wp-upload/MONACO.TTF
	}
	.each do |host_and_path|
		download "http://#{host_and_path}", "#{BUILD_DIR}/install/#{File.basename(host_and_path)}"
	end
end

def install_gems
	Bundler.with_clean_env do
		system("#{BUILD_DIR}/set-env.bat \
			&& gem uninstall vagrant -a -x -I \
			&& gem install bundler --no-ri --no-rdoc \
			&& bundle install --gemfile=#{BUILD_DIR}/Gemfile --verbose")

		# XXX: need to re-install/override the :git gems as bundler does not install them properly
		# (i.e. they won't show up in `gem list` neither will the binaries by installed)
		[
			%w{ vagrant 	1.0.2.patch1 },
			%w{ sahara 		0.0.10.patch1 }
		]
		.each do |name, version|
			clone_dir = `#{BUILD_DIR}/set-env.bat && cd #{BUILD_DIR} && bundle show #{name}`.split.last
			system("#{BUILD_DIR}/set-env.bat \
				&& cd #{clone_dir} \
				&& gem build #{name}.gemspec \
				&& gem install #{name}-#{version}.gem --no-ri --no-rdoc \
				&& copy #{name}-#{version}.gem #{BUILD_DIR}/install")			
		end
	end
end

def clone_repositories
	[ 
		%w{ npverni/cucumber-sublime2-bundle.git 	tools/sublimetext2/Data/Packages/Cucumber },
		%w{ cabeca/SublimeChef.git 					tools/sublimetext2/Data/Packages/Chef },
		%w{ tknerr/bills-kitchen-repo.git 			repo/my-chef-repo },
		%w{ tknerr/vagrant-ohai.git 				repo/my-cookbooks/vagrant-ohai },
		%w{ tknerr/cookbooks-motd.git 				repo/my-cookbooks/motd },
		%w{ tknerr/vagrant-baseboxes.git 			repo/my-baseboxes }
	]
	.each do |repo, dest|
		system("git clone git://github.com/#{repo} #{BUILD_DIR}/#{dest}")
	end
end

def assemble_kitchen
	pack BUILD_DIR, "#{TARGET_DIR}/bills-kitchen-#{VERSION}.7z"
end


def download_and_unpack(url, target_dir, includes = [])	
	Dir.mktmpdir do |tmp_dir| 
		outfile = "#{tmp_dir}/#{File.basename(url)}"
		download(url, outfile)
		unpack(outfile, target_dir, includes)
	end
end

def download(url, outfile)
	puts "checking cache for '#{url}'"
	url_hash = Digest::MD5.hexdigest(url)
	cached_file = "#{CACHE_DIR}/#{url_hash}"
	if File.exist? cached_file
		puts "cache-hit: read from '#{url_hash}'"
		FileUtils.cp cached_file, outfile
	else
		download_no_cache(url, outfile)
		puts "caching as '#{url_hash}'"
		FileUtils.cp outfile, cached_file
	end
end

def download_no_cache(url, outfile)
	puts "download '#{url}'"
	uri = URI(url)
	Net::HTTP.start(uri.host, uri.port) do |http|
		File.open(outfile, 'wb') do |f|
			http.request_get(uri.path + (uri.query ? "?#{uri.query}" : '')) do |resp|
				resp.read_body do |segment|
					f.write(segment)
				end
			end
		end
	end
end

def unpack(archive, target_dir, includes = [])
	puts "extracting '#{archive}' to '#{target_dir}'"	
	case File.extname(archive)
	when '.zip', '.7z'
		system("\"#{ZIP_EXE}\" x -o\"#{target_dir}\" -y \"#{archive}\" 1> NUL")
	when '.exe'
		system("\"#{ZIP_EXE}\" e -o\"#{target_dir}\" -y \"#{archive}\" -r #{includes.join(' ')} 1> NUL")
	when '.msi'
		system("start /wait msiexec /a \"#{archive.gsub('/', '\\')}\" /qb TARGETDIR=\"#{target_dir.gsub('/', '\\')}\"")
	else 
		raise "don't know how to unpack '#{archive}'"
	end
end

def pack(target_dir, archive)
	puts "packing '#{target_dir}' into '#{archive}'"
	system("\"#{ZIP_EXE}\" a -t7z -y \"#{archive}\" \"#{target_dir}\" 1> NUL")
end