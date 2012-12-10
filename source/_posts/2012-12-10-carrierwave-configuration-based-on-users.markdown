---
layout: post
title: "Carrierwave configuration based on users"
date: 2012-12-10 16:54
comments: true
categories:
- Carrierwave
- Ruby on Rails
- Ruby
---
So, you have an multitenant application that allows users to upload files, which are tied to their accounts. The users own these files, so, they should not be sitting in on large bucket with everyone else's files. The files should be by themselves.

Carrierwave removes many of the complications involved with dealing with file uploads in rails, from thumbnails, to extension whitelisting, it has you covered.

We have one problem, though. Their documentation only shows how to configure settings inside of a Rails initializer. This does not allow us to have user configurable settings. The application has to be restarted to alter any setting. This is possible, though. To quote the documentation, "CarrierWave has a broad range of configuration options, which you can configure, both globally and on a **per-uploader basis**"

So, how do we do this? Simple. All carrierwave configureation options are methods within the uploader object:

``` ruby lib/carrierwave/uploader/configuration.rb
module CarrierWave
  module Uploader
    module Configuration
      extend ActiveSupport::Concern

      included do
        class_attribute :_storage, :instance_writer => false

        add_config :root
        add_config :base_path
        add_config :asset_host
        add_config :permissions
        add_config :directory_permissions
        # ...

        # fog
        add_config :fog_attributes
        add_config :fog_credentials
        add_config :fog_directory
        add_config :fog_endpoint
        add_config :fog_public
        add_config :fog_authenticated_url_expiration

        # ...
```
[lib/carrierwave/uploader/configuration.rb](https://github.com/jnicklas/carrierwave/blob/master/lib/carrierwave/uploader/configuration.rb)

We can then see where each option's methods are being created.

``` ruby lib/carrierwave/uploader/configuration.rb:79
def add_config(name)
  class_eval <<-RUBY, __FILE__, __LINE__ + 1
    def self.#{name}(value=nil)
      @#{name} = value if value
      return @#{name} if self.object_id == #{self.object_id} || defined?(@#{name})
      name = superclass.#{name}
      return nil if name.nil? && !instance_variable_defined?("@#{name}")
      @#{name} = name && !name.is_a?(Module) && !name.is_a?(Symbol) && !name.is_a?(Numeric) && !name.is_a?(TrueClass) && !name.is_a?(FalseClass) ? name.dup : name
    end

    def self.#{name}=(value)
      @#{name} = value
    end

    def #{name}=(value)
      @#{name} = value
    end

    def #{name}
      value = @#{name} if instance_variable_defined?(:@#{name})
      value = self.class.#{name} unless instance_variable_defined?(:@#{name})
      value.instance_of?(Proc) ? value.call : value
    end
  RUBY
end
```
[lib/carrierwave/uploader/configuration.rb Line 79](https://github.com/jnicklas/carrierwave/blob/master/lib/carrierwave/uploader/configuration.rb#L79)

Talk aside, lets build a simple uploader class that pulls the settings from our user class.

> This example assumes that the resource ```belongs_to :user```.

``` ruby app/uploaders/resource_uploader.rb
class ResourceUploader < CarrierWave::Uploader::Base
  storage :fog

  # Override configuration
  def fog_credentials
    {
      provider: 'Rackspace',
      rackspace_username: model.owner.cloudfiles_username,
      rackspace_api_key: model.owner.cloudfiles_apikey
    }
  end

  def fog_directory
    model.owner.rackspace_container_name
  end
end
```

The object ```model``` will give you access to the model inwhich the uploader is mounted within.

I hope this helps anyone who is just begining to dive down this road. I will provide more details on this as my code utilizing this concept matures.
