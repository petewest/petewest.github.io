---
layout: post
title:  "Hosting your own Chef"
date:   2014-09-12 19:40:00
categories: devops
---

[Chef][] seems to be a pretty decent tool but it took me a while to get a working server &
client set up on version 12 due to the merging of the old Enterprise and Open Source editions
that were available in version 11, the subtle differences between the two, and the lack of 
documentation currently available for version 12.

Here's how I managed to get it set up and working.

## Environment

### Server

- OS: ubuntu-14.04.1-server
- Hostname: chefserver
- Chef: chef-server-core_12.0.0-rc.3-1
- HDD: 40 GB

### Workstation

- OS: ubuntu-14.04.1-server
- Hostname: chefwork
- Chef: chefdk-0.2.1-1
- HDD: 20 GB

### Variables

Wherever you see a value in < > below (e.g. `<username>`) replace it your own information.


## Server

1. Log in to your fresh server VM
2. Download the [server package][download_server]
3. Become root: `sudo -s`
4. Install the package: `dpkg -i chef-server*.deb`
5. Run the initial configuration: `chef-server-ctl reconfigure`
6. Create your organisation and save the certificate to a file:

		chef-server-ctl org-create <organisation> "<Long name in quotes>" -f <organisation>-validator.pem

7. Create your user and save the certificate to a file:

		chef-server-ctl user-create <username> <FirstName> <LastName> <email> <password> -f <username>.pem

8. Associate your new user with your new organisation:

		chef-server-ctl org-associate <organisation> <username>

Your server should now be set up, running, and have a user that can start adding nodes, etc.
As we used sudo your `.pem` files are owned by root, so you might want to tidy those up:

	chown <username>:<username> <username>.pem <organisation>-validator.pem


## Workstation

1. Log in to your fresh workstation VM
2. Download the [development kit][dev_kit]
3. Install the package: `sudo dpkg -i chefdk*.deb`
4. Create a folder for all your chef files: `chef generate repo chef-repo`
5. We need to create a config file and populate it with data.
	This assumes your chef `<username>` set above matches your workstation username, if not,
	change the ``chef_user = `whoami`.chomp`` line below to `chef_user = '<username>'`

		cd chef-repo
		mkdir .chef

6. Copy your certificates from the server to your new folder:

		scp chefserver:~/*.pem .
		chmod go-rwx *.pem

7. If you're adding your `chef-repo` to source control remember to
	add an ignore rule for `.chef/*.pem` (e.g.
	`echo '.chef/*.pem' >> ~/chef-repo/.gitignore` for git)
8. With your favourite text editor, create a file called `knife.rb`
	with the following (not forgetting to replace `<organisation>`):

{% highlight ruby %}
chef_user = `whoami`.chomp
current_folder = File.dirname(__FILE__)
organisation = '<organisation>'
log_level :info
log_location STDOUT
node_name chef_user
client_key "#{current_folder}/#{chef_user}.pem"
validation_client_name "#{organisation}-validator"
validation_client_key "#{current_folder}/#{organisation}-validator.pem"
chef_server_url "https://chefserver/organizations/#{organisation}"
cookbook_path ["#{current_folder}/../cookbooks"]
{% endhighlight %}

Test your install by listing the users and nodes on your new server:
	
	cd ..
	knife user list
	knife node list

The user list should show your `<username>`, and the node list should
currently be blank, but neither should error.

That's it!  Your server and workstation are now configured and you can carry on with the [tutorials][]


[Chef]: http://getchef.com
[download_server]: http://downloads.getchef.com/chef-server/ubuntu/
[dev_kit]: http://downloads.getchef.com/chef-dk/ubuntu/
[tutorials]: http://learn.getchef.com/
