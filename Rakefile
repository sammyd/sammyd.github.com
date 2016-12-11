require 'html-proofer'

task :test do
  options = {
    allow_hash_href: true,
    assume_extension: true,
    check_favicon: true,
    check_opengraph: false,
    only_4xx: true
  }

  HTMLProofer.check_directory("./_site", options).run
end
