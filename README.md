# databasedotcom

[![Build Status](https://travis-ci.org/heroku/databasedotcom.png?branch=master)](https://travis-ci.org/heroku/databasedotcom) [![Code Climate](https://codeclimate.com/badge.png)](https://codeclimate.com/github/heroku/databasedotcom) [![Dependency Status](https://gemnasium.com/heroku/databasedotcom.png)](https://gemnasium.com/heroku/databasedotcom)

databasedotcom is a gem to enable ruby applications to access the SalesForce REST API. 
If you use bundler, simply list it in your Gemfile, like so:

```
gem 'databasedotcom', :git=>"git@github.com:phongleland/databasedotcom-kb_articles.git"
```

If you don't use bundler, install it by hand:

```
gem install databasedotcom, :git=>"git@github.com:phongleland/databasedotcom-kb_articles.git"
```

## Documentation

Reference documentation is available at [rubydoc.info](http://rubydoc.info/github/heroku/databasedotcom/master/frames)

## Source

Source is available at [github](http://github.com/heroku/databasedotcom)

## Contributions

To contribute, fork this repo, make changes in your fork, then send a pull request. 
No pull requests without accompanying tests will be accepted.  To run tests in your
fork, just do 

```
bundle install
rake
```

# Usage
## Initialization
When you create a Databasedotcom::Client object, you need to configure it with a client
id and client secret that corresponds to one of the Remote Access Applications configured
within your Salesforce instance.  The Salesforce UI refers to the client id as "Consumer Key",
and to the client secret as "Consumer Secret".

You can configure your Client object with a client id and client secret in one of several
different ways:

### Configuration from the environment

If configuration information is present in the environment, the new Client will take configuration
information from there.

```bash
export DATABASEDOTCOM_CLIENT_ID=foo
export DATABASEDOTCOM_CLIENT_SECRET=bar
```

Then

```ruby
client = Databasedotcom::Client.new
client.client_id      #=> foo
client.client_secret  #=> bar
```

### Configuration from a YAML file

If you pass the name of a YAML file when you create a Client, the new Client will read the YAML
file and take the client id and client secret values from there.

```yaml
# databasedotcom.yml
#
---
client_secret: bro
client_id: baz
```

Then

```ruby
client = Databasedotcom::Client.new("databasedotcom.yml")
client.client_id      #=> bro
client.client_secret  #=> baz
```

### Configuration from a Hash

If you pass a hash when you create a Client, the new Client will take configuration information
from that Hash.

```ruby
client = Databasedotcom::Client.new :client_id => "sponge", :client_secret => "bob"
client.client_id      #=> sponge
client.client_secret  #=> bob
```

### Configuration precedence

Configuration information present in the environment always takes precedence over that passed in
via a YAML file or a Hash.

```bash
export DATABASEDOTCOM_CLIENT_ID=foo
export DATABASEDOTCOM_CLIENT_SECRET=bar
```

Then

```ruby
client = Databasedotcom::Client.new :client_id => "sponge", :client_secret => "bob"
client.client_id      #=> foo
client.client_secret  #=> bar
```

### Usage in an application deployed on Heroku

You can use the `heroku config:add` command to set environment variables:

```bash
heroku config:add DATABASEDOTCOM_CLIENT_ID=foo
heroku config:add DATABASEDOTCOM_CLIENT_SECRET=bar
```

Then, when you create your client like:

```ruby
client = Databasedotcom::Client.new
```

it will use the configuration information that you set with `heroku config:add`.

### Connect to a SalesForce sandbox account

Specify the `:host` option when creating your Client, e.g,

```ruby
Databasedotcom::Client.new :host => "test.salesforce.com", ...
```

## Authentication

The first thing you need to do with the new Client is to authenticate with Salesforce. 
You can do this in one of several ways:

### Authentication via an externally-acquired OAuth access token

If you have acquired an OAuth access token for your Salesforce instance through some external
means, you can use it.  Note that you have to pass both the token and your Salesforce instance
URL to the `authenticate` method:

```ruby
client.authenticate :token => "my-oauth-token", :instance_url => "http://na1.salesforce.com"  #=> "my-oauth-token"
```

### Authentication via Omniauth

If you are using the gem within the context of a web application, and your web app is using Omniauth
to do OAuth with Salesforce, you can authentication the Client direction via the Hash that Omniauth
passes to your OAuth callback method, like so:

```ruby
client.authenticate request.env['omniauth.auth']  #=> "the-oauth-token"
```

### Authentication via username and password

You can authenticate your Client directly with Salesforce with a valid username and password for
a user in your Salesforce instance.  Note that, if access to your Salesforce instance requires a
[security token](http://www.salesforce.com/us/developer/docs/api/Content/sforce_api_concepts_security.htm),
the value that you pass for <tt>:password</tt> must be the password for the user concatenated with
her security token.

```ruby
client.authenticate :username => "foo@bar.com", :password => "ThePasswordTheSecurityToken"  #=> "the-oauth-token"
```

## Editing KB Articles

Before you can edit an article you need to create a draft

```ruby
client.create_draft(f.KnowledgeArticleId)
```

Query for the drafts of the KB articles

```ruby
drafts=client.query("select id from FAQ__kav where PublishStatus='Draft' AND Language='en_US' and KnowledgeArticleId='#{f.KnowledgeArticleId}'")
```

The article you just created is the last of the array object
      
```ruby
draft=drafts.last
```

Now you can edit as you would rails-style

```ruby
draft.update_attributes({:Security__c=>"Premium"})
```

Finally you will need to publish your draft

```ruby
client.publish_article(draft.Id)
```

## Accessing the Sobject API

You can retrieve a list of Sobject defined in your Salesforce instance like so:

```ruby
client.list_sobjects  #=> ['User', 'Group', 'Contact']
```

Once you have the name of an Sobject, the easiest way to interact with it is to first materialize it:

```ruby
contact_class = client.materialize("Contact") #=> Contact
```

By default, Sobject classes are materialized into the global namespace- if you want materialize into
another module, you can easily do configure this:

```ruby
client.sobject_module = My::Module
client.materialize("Contact") #=> My::Module::Contact
```

Materialized Sobject classes behave much like ActiveRecord classes:

```ruby
contact = Contact.find("contact_id")                #=> #<Contact @Id="contact_id", ...>
contact = Contact.find_by_Name("John Smith")        #=> dynamic finders!
contacts = Contact.all                              #=> a Databasedotcom::Collection of Contact instances
contacts = Contact.find_all_by_Company("IBM")       #=> a Databasedotcom::Collection of matching Contacts
contact.Name                                        #=> the contact's Name attribute
contact["Name"]                                     #=> same thing
contact.Name = "new name"                           #=> change the contact's Name attribute, in memory
contact["Name"] = "new name"                        #=> same thing
contact.save                                        #=> save the changes to the database
contact.update_attributes "Name" => "newer name",
  "Phone" => "4156543210"                           #=> change several attributes at once and save them
contact.delete                                      #=> delete the contact from the database
```

See the [documentation](http://rubydoc.info/github/heroku/databasedotcom/master/frames) for full details.

## Accessing the Chatter API

You can easily access Chatter feeds, group, conversations, etc.:

```ruby
my_feed_items = Databasedotcom::Chatter::UserProfileFeed.find(client)  #=> a Databasedotcom::Collection of FeedItems

my_feed_items.each do |feed_item|
  feed_item.likes                   #=> a Databasedotcom::Collection of Like instances
  feed_item.comments                #=> a Databasedotcom::Collection of Comment instances
  feed_item.raw_hash                #=> the hash returned from the Chatter API describing this FeedItem
  feed_item.comment("This is cool") #=> create a new comment on the FeedItem
  feed_item.like                    #=> the authenticating user likes the FeedItem
end

me = Databasedotcom::Chatter::User.find(client, "me")   #=> a User for the authenticating user
me.followers                                              #=> a Databasedotcom::Collection of Users
me.post_status("what I'm doing now")                      #=> post a new status

you = Databasedotcom::Chatter::User.find(client, "your-user-id")
me.follow(you)                                            #=> start following a user
```

See the [documentation](http://rubydoc.info/github/heroku/databasedotcom/master/frames) for full details.

# License

This gem is licensed under the MIT License.
