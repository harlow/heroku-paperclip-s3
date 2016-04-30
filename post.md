> WARNING: The following code samples are not compatible with the new AWS 2.0 SDK. We are in the process of updating the text and example repository.

Many web apps require the user to upload images and other files for storage and processing. Paperclip is a cleanly abstracted Ruby library that reduces the complexity of file uploading and processing.

Using Paperclip with an external storage service such as Amazon S3 or Rackspace CloudFiles allows you to scale your application's files and codebase independently. Such an architecture is required in highly elastic environments that distribute your application across multiple instances, such as Heroku.

This guide describes how to set up a Ruby on Rails application with image uploading via Paperclip and Amazon S3.

## Prerequisites

* [AWS S3 Account](https://devcenter.heroku.com/articles/s3#s3-setup) for storing images in the cloud.
* [Heroku Toolbelt](https://toolbelt.heroku.com/) to create and deploy web applications to Heroku.
* [ImageMagick](http://www.imagemagick.org/script/index.php) for resizing images. 

Note: Mac users can install ImageMagick with [Homebrew](http://mxcl.github.com/homebrew/) `brew install imagemagick`. Windows users can use the [Windows binary release](http://www.imagemagick.org/script/binary-releases.php#windows).

## Overview

Paperclip is an easy file attachment library for `ActiveRecord`. It treats files like model attributes. This means they aren't saved to their final locations, nor are they deleted if set to `nil`, until `ActiveRecord::Base#save` is called. It can validate based on file size and presence. It can transform its assigned image into thumbnails if needed, and the only prerequisites are database columns and ImageMagick. Attached files are referenced in the browser by an understandable specification with sensible defaults.

## Reference Application

![Paperclip Demo Application](https://s3.amazonaws.com/heroku-devcenter-files/article-images/841-imported-1443570312-841-imported-1443554781-paperclip_demo_screenshot_470.png)

The reference application allows a user to manage a list of their friends. 

* Each friend will have an `avatar` with Paperclip providing the image upload and resizing functionality. 
* The app will demonstrate how to generate scaled down thumbnails, and display the resized images. 
* The application will also gracefully degrade to display a default image `missing.png` for friends without an avatar.

<a href="https://github.com/thoughtbot/paperclip_demo" target="_blank">Download the source code from GitHub</a>.

## Configuration

Paperclip requires the following gems added to your Gemfile.

```ruby
# Gemfile
gem 'paperclip'
gem 'aws-sdk'
```

Run `bundle install` and restart the Rails server after modifying the Gemfile.

We'll also need to specify the AWS configuration variables for the production Environment.

```ruby
# config/environments/production.rb
config.paperclip_defaults = {
  :storage => :s3,
  :s3_credentials => {
    :bucket => ENV['S3_BUCKET_NAME'],
    :access_key_id => ENV['AWS_ACCESS_KEY_ID'],
    :secret_access_key => ENV['AWS_SECRET_ACCESS_KEY']
  }
}
```

<p class="callout">
To test S3 uploads locally in development mode these settings must also be added to the <a href="https://github.com/thoughtbot/paperclip_demo/blob/master/config/environments/development.rb" target="_blank">development environment</a>.
</p>

Additionally, we'll need to the set the AWS configuration variables on the Heroku application.

```terminal
$ heroku config:set S3_BUCKET_NAME=your_bucket_name
$ heroku config:set AWS_ACCESS_KEY_ID=your_access_key_id
$ heroku config:set AWS_SECRET_ACCESS_KEY=your_secret_access_key
```

## International users (additional configuration)

If you are having issues uploading images please read the following two configuration sections.

<p class="note">
 If you continue to have issues please see the <a href="http://rdoc.info/github/thoughtbot/paperclip/Paperclip/Storage/S3" target="_blank">Paperclip documentation</a> page for detailed configuration options.
</p>

To override the default URL structure and place the bucket's name  "domain-style" in the URL (e.g. your_bucket_name.s3.amazonaws.com). These options can be placed in the  `paperclip_defaults` configuration hash shown above, or into an initializer. 

```ruby
# config/initializers/paperclip.rb 
Paperclip::Attachment.default_options[:url] = ':s3_domain_url'
Paperclip::Attachment.default_options[:path] = '/:class/:attachment/:id_partition/:style/:filename'
```

If you are seeing the following error: "The bucket you are attempting to access must be addressed using the specified endpoint. Please send all future requests to this endpoint." Try setting the specified endpoint with the `s3_host_name` config var.

```ruby
# config/initializers/paperclip.rb 
Paperclip::Attachment.default_options[:s3_host_name] = 's3-us-west-2.amazonaws.com'
```

## Define the file attribute in the Model

To add attachment functionality to the `Friend` [model](https://github.com/thoughtbot/paperclip_demo/blob/master/app/models/friend.rb) use the Paperclip helper method `has_attached_file` and a symbol with the desired name of the attachment.

```ruby
class Friend < ActiveRecord::Base
  # This method associates the attribute ":avatar" with a file attachment
  has_attached_file :avatar, styles: {
    thumb: '100x100>',
    square: '200x200#',
    medium: '300x300>'
  }

  # Validate the attached image is image/jpg, image/png, etc
  validates_attachment_content_type :avatar, :content_type => /\Aimage\/.*\Z/
end
```

<div class="callout">
Additional ImageMagick resizing options <a href="http://www.imagemagick.org/Usage/resize/" target="_blank">can be found here</a>
</div>

The `has_attached_file` method also accepts a `styles` hash that specifies the resize dimensions of the uploaded image. The `>` and `#` symbols will tell ImageMagick how the image will be resized (the `>` will proportionally reduce the size of the image).

## Update database

A database migration is needed to add the `avatar` attribute on `Friend` in the database schema. Run the following rails helper method to generate a stub migration.

```ruby
$ rails g migration AddAvatarToFriends
```

Paperclip comes with the migration helper methods `add_attachment` and `remove_attachment`. They are used to create the columns needed to store image data in the database. Use them in the `AddAvatarToFriends` migration.

```ruby
class AddAvatarToFriends < ActiveRecord::Migration
  def self.up
    add_attachment :friends, :avatar
  end

  def self.down
    remove_attachment :friends, :avatar
  end
end
```

[This migration](https://github.com/thoughtbot/paperclip_demo/blob/master/db/migrate/20120802191046_add_avatar_to_friends.rb)  will create `avatar_file_name`, `avatar_file_size`, `avatar_content_type`, and `avatar_updated_at` attributes on the `Friend` model. These attributes will be set automatically when files are uploaded.

Run the migrations with `rake db:migrate` to update your database.
 
## Upload form

Images are uploaded to your application before being stored in S3. This allows your models to perform validations and other processing before being sent to S3.

![Form with File Input](https://s3.amazonaws.com/heroku-devcenter-files/article-images/841-imported-1443570313-841-imported-1443554782-paperclip_form_470.png)
   
Add a file input field to the [web form](https://github.com/thoughtbot/paperclip_demo/blob/master/app/views/friends/_form.html.erb) that allows users to browse and select images from their local filesystem. 

Make sure the form has `multipart: true` added to it.

```rhtml
<%= form_for(@friend, multipart: true) do |f| %>
  <div class="field">
    <%= f.label :name %>
    <%= f.text_field :name %>
  </div>
  <div class="field">
    <%= f.label :avatar %>
    <%= f.file_field :avatar %>
  </div>
  <div class="actions">
    <%= f.submit 'Make a friend' %>
    <%= link_to 'Nevermind', friends_path, class: 'button' %>
  </div>
<% end %>
```

When the form is submitted and the backing models are successfully persisted to the database, the file itself will be uploaded to S3.

## Upload controller

With Rails 4 we'll need to specify the permitted params. We'll permit `:name` and `:avatar` in the params. 

```ruby
class FriendsController < ApplicationController
  # Other CRUD actions omitted

  def create
    @friend = Friend.new(friend_params)

    if @friend.save
      redirect_to @friend, notice: 'Friend was successfully created.'
     else
       render action: 'new'
    end
  end

  private

  def friend_params
    params.require(:friend).permit(:avatar, :name)
  end
end
```

<p class="warning">
Large files uploads in single-threaded, non-evented environments (such as Rails) block your application’s web dynos and can cause request timeouts and <a href="https://devcenter.heroku.com/articles/error-codes#h11-backlog-too-deep" target="_blank">H11, H12 errors</a>. For files larger than 4mb the <a href="https://devcenter.heroku.com/articles/s3#direct-upload" target="_blank">direct upload</a> method should be used instead.
</p>

## Image display

Files that have been uploaded with Paperclip are stored in S3. However, metadata such as the file's name, location on S3, and last updated timestamp are all stored in the model's table in the database.

Access the file's url through the `url` method on the model's file attribute (`avatar` in this example).

```ruby
friend.avatar.url #=> http://your_bucket_name.s3.amazonaws.com/...
```

This url can be used directly in the view to [display uploaded images](https://github.com/thoughtbot/paperclip_demo/blob/master/app/views/friends/_friend.html.erb).

```ruby
<%= image_tag @friend.avatar.url(:square) %>
```

The `url` method can take a style (defined earlier in the `Friend` model) to access a specific processed version of the file.

![Paperclip Demo Application](https://s3.amazonaws.com/heroku-devcenter-files/article-images/841-imported-1443570313-841-imported-1443554782-paperclip_grid_view_470.png)

As these images are served directly from S3 they don't interfere with other requests and allow your page to load quicker than serving directly from your app.

Display the medium sized image by passing `:medium` to the `url` method.

```rhtml
<%= image_tag @friend.avatar.url(:medium) %>
```

## Deploy

Once you've updated your application to use Paperclip commit the modified files to git.

```term
$ git commit -m "Upload friend images via Paperclip"
```

On deployment to Heroku you will need to migrate your database to support the required file columns.

```term
$ git push heroku master
$ heroku run bundle exec rake db:migrate
```
