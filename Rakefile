require 'html-proofer'

task :test do
  sh "bundle exec jekyll build"

  options = {
    allow_hash_href: true,
    assume_extension: true,
    typhoeus: {
      ssl_verifypeer: false
    },
    check_favicon: true,
    check_opengraph: false,
    only_4xx: true
  }

  HTMLProofer.check_directory("./_site", options).run
end

task :deploy do
  # TODO
end
