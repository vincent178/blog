title: How to run localhost rails on https
date: 2015-10-18 20:52:15
tags:
---

### First Step

```
openssl req -new -newkey rsa:2048 -sha1 -days 365 -nodes -x509 -keyout server.key -out server.crt
```

### Second Step

```
~ mkdir certs
~ mv server.crt server.key certs
```

### Third Step

modify `bin/rails` to file like follows
```
#!/usr/bin/env ruby

require 'rails/commands/server'
require 'rack'
require 'webrick'
require 'webrick/https'

if ENV['SSL'] == "true"
  module Rails
      class Server < ::Rack::Server
          def default_options
              super.merge({
                  :Port => 3001,
                  :environment => (ENV['RAILS_ENV'] || "development").dup,
                  :daemonize => false,
                  :debugger => false,
                  :pid => File.expand_path("tmp/pids/server.pid"),
                  :config => File.expand_path("config.ru"),
                  :SSLEnable => true,
                  :SSLVerifyClient => OpenSSL::SSL::VERIFY_NONE,
                  :SSLPrivateKey => OpenSSL::PKey::RSA.new(
                                   File.open("certs/server.key").read),
                  :SSLCertificate => OpenSSL::X509::Certificate.new(
                                   File.open("certs/server.crt").read),
                  :SSLCertName => [["CN", WEBrick::Utils::getservername]],
              })
          end
      end
  end
end


APP_PATH = File.expand_path('../../config/application', __FILE__)
require_relative '../config/boot'
require 'rails/commands'
```

### Last
```
~ SSL=true rails s webrick
```
