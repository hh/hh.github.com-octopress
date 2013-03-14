---
layout: post
title: "Vagrant van down by the river... with fresh ingredients!"
date: 2013-03-14 12:35
comments: true
categories: 
- vagrant
- veewee
- ii
---
I was super excited to see the release of [Vagrant 1.1.0 from Hashicorp](http://www.hashicorp.com/blog/vagrant-1-1-and-vmware.html)! I immediately set about trying out this new ingredient in my instant infrastructure USB deployer.

First step was to use my [source_ingredient_vagrant](https://github.com/easybake-ingredients/vagrant/blob/master/.chef/plugins/knife/source_ingredient_vagrant.rb) knife plugin to auto download any new versions that might be available into my :file_cache_path and create some data_bag/json files describing the ingredients. The plugin automatically found the new releases for me and updated my data bags!


## knife source ingredient vagrant

```
[hh@M18xR2:~/easybake/oven/repos/cloud-kitchen (master)✗]
☺ ➔  knife source ingredient vagrant
Downloading http://files.vagrantup.com/packages/194948999371e9aee391d13845a0bdeb27e51ac0/Vagrant.dmg to /home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/Vagrant-v1.1.0.dmg
Downloading http://files.vagrantup.com/packages/194948999371e9aee391d13845a0bdeb27e51ac0/Vagrant.msi to /home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/Vagrant-v1.1.0.msi
... snip ...
Downloading http://files.vagrantup.com/packages/194948999371e9aee391d13845a0bdeb27e51ac0/vagrant_x86_64.rpm to /home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/vagrant-v1.1.0_x86_64.rpm 
[hh@M18xR2:~/easybake/oven/repos/cloud-kitchen (master)✗]
☺ ➔  ls .chef/cache/*1.1.0*
.chef/cache/Vagrant-v1.1.0.dmg       .chef/cache/Vagrant-v1.1.0.msi
.chef/cache/vagrant-v1.1.0_i686.deb  .chef/cache/vagrant-v1.1.0_x86_64.deb
.chef/cache/vagrant-v1.1.0_i686.rpm  .chef/cache/vagrant-v1.1.0_x86_64.rpm
[hh@M18xR2:~/easybake/oven/repos/cloud-kitchen (master)✗]
☹ ➔  ls data_bags/vagrant/*1_1_0*
data_bags/vagrant/el_i686_v1_1_0.json
data_bags/vagrant/el_x86_64_v1_1_0.json
data_bags/vagrant/osx_v1_1_0.json
data_bags/vagrant/ubuntu_i686_v1_1_0.json
data_bags/vagrant/ubuntu_x86_64_v1_1_0.json
data_bags/vagrant/windows_v1_1_0.json
```

The resultant data_bag jason files can be seen at [ingredients.easybake.cd](http://ingredients.easybake.cd) or specifically:

[https://github.com/easybake-ingredients/vagrant](https://github.com/easybake-ingredients/vagrant)

I *could* upload those vagrant data_bag/ingredients to a chef server, but for now I just add chef-solo-search to my list of recipes to run. **It would be really interesting that if used as part of a continuous delivery process, imagine automatically integrate third party ingredients as they update...**

## [easybake-workstation](https://github.com/easybake-cookbooks/easybake-workstation)

In trying to easily bake a devops workstation, I set some attributes to specify which externally sourced ingredients ingredients I want to include... and optionally the pinned semantic versions:

```ruby easybake-workstation/attributes/default.rb https://github.com/easybake-cookbooks/easybake-workstation/blob/master/attributes/default.rb#L5
default['easybake-workstation']['ingredients']['vagrant']['desc']='Vagrant'
```
By default we use the latest version of each ingredient:

```ruby easybake-workstation/recipes/ingredients.rb https://github.com/easybake-cookbooks/easybake-workstation/blob/master/recipes/ingredients.rb
node['easybake-workstation']['ingredients'].each do |data_bag,attrs|
  all_artifacts = search(data_bag,
  "os_#{node.platform}:#{node.platform_version} AND arch:#{node.kernel.machine}")
  node.default['easybake-workstation']['ingredients'][data_bag]['version']=\
    all_artifacts.map{|v| v['version']}.flatten.uniq.sort.last
```

We then search for the desired ingredient and ensure it's in the cache, and eventually set the path to the local file for it.For vagrant we just install the package specified by the ingredients file attribute.

```ruby easybake-workstation/recipes/ingredients.rb https://github.com/easybake-cookbooks/easybake-workstation/blob/master/recipes/ingredients.rb
 query = "version:#{node['easybake-workstation']['ingredients'][data_bag]['version']}"
 search(data_bag,query).each do |ingredient|
    cache_file = File.join(Chef::Config[:file_cache_path], ing['filename'])
    # ... ensure cache is populated
    node.default['easybake-workstation']['ingredients'][data_bag]['file']=cache_file
end
```

For vagrant we just install the package specified by the ingredient attribute.

```ruby easybake-workstation/recipes/vagrant.rb https://github.com/easybake-cookbooks/easybake-workstation/blob/master/recipes/vagrant.rb
dpkg_package 'vagrant' do
  source node['easybake-workstation']['ingredients']['vagrant']['file']
end
```

## Vagrant as an automatically updated ingredient

Obviously there is some work to be done to the easybake-workstation to make it work on all supported platforms, chef ingredients at this point are really conceptual, but I hope this real world example helps generate some discussions.

```bash
[hh@M18xR2:~/easybake/oven/repos/cloud-kitchen (master)✗] 
☺ ➔  sudo chef-solo -c .chef/solo.rb 
/home/hh/easybake/oven/repos/cloud-kitchen/.chef
Starting Chef Client, version 11.4.0
Compiling Cookbooks...
Recipe: easybake-workstation::ingredients
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/7601.17514.101119-1850_x64fre_server_eval_en-us-GRMSXEVAL_EN_DVD.iso] action create (skipped due to not_if)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/7601.17514.101119-1850_x64fre_server_eval_en-us-GRMSXEVAL_EN_DVD.iso.checksum] action create (up to date)
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/virtualbox-4.2_4.2.8-83876~Ubuntu~precise_amd64.deb] action create (skipped due to not_if)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/virtualbox-4.2_4.2.8-83876~Ubuntu~precise_amd64.deb.checksum] action create (up to date)
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/vagrant-v1.1.0_x86_64.deb] action create (up to date)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/vagrant-v1.1.0_x86_64.deb.checksum] action create
    - create new file /home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/vagrant-v1.1.0_x86_64.deb.checksum with content checksum 09dc73
        --- /tmp/chef-tempfile20130314-31126-1cpzs0a 2013-03-14 11:23:44.030889110 -0700
        +++ /tmp/chef-diff20130314-31126-1lw8m9t 2013-03-14 11:23:44.026889064 -0700
        @@ -0,0 +1 @@
        +70024b642757517f7fc0747f736055f8b3e2febe669c779addfb6a014c848c79
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/Sublime Text 2.0.1 x64.tar.bz2] action create (skipped due to not_if)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/Sublime Text 2.0.1 x64.tar.bz2.checksum] action create (up to date)
Converging 46 resources
Recipe: easybake-workstation::default
  * package[git] action install (up to date)
  * package[g++] action install (up to date)
  * package[libxml2-dev] action install (up to date)
  * package[libxslt-dev] action install (up to date)
Recipe: easybake-workstation::ingredients
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/7601.17514.101119-1850_x64fre_server_eval_en-us-GRMSXEVAL_EN_DVD.iso] action create (skipped due to not_if)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/7601.17514.101119-1850_x64fre_server_eval_en-us-GRMSXEVAL_EN_DVD.iso.checksum] action create (up to date)
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/virtualbox-4.2_4.2.8-83876~Ubuntu~precise_amd64.deb] action create (skipped due to not_if)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/virtualbox-4.2_4.2.8-83876~Ubuntu~precise_amd64.deb.checksum] action create (up to date)
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/vagrant-v1.1.0_x86_64.deb] action create (skipped due to not_if)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/vagrant-v1.1.0_x86_64.deb.checksum] action create (up to date)
  * remote_file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/Sublime Text 2.0.1 x64.tar.bz2] action create (skipped due to not_if)
  * file[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/Sublime Text 2.0.1 x64.tar.bz2.checksum] action create (up to date)
Recipe: easybake-workstation::virtualbox
  * dpkg_package[virtualbox] action install (up to date)
Recipe: easybake-workstation::vagrant
  * dpkg_package[vagrant] action install
    - install version 1.1.0 of package vagrant
  * gem_package[vagrant-windows] action install
    - install version 0.1.2 of package vagrant-windows
  * gem_package[em-winrm] action install
    - install version 0.5.4 of package em-winrm
  * gem_package[knife-windows] action install (up to date)
  * git[/home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/veewee-gem] action sync
    - clone from git://github.com/jedi4ever/veewee.git into /home/hh/easybake/oven/repos/cloud-kitchen/.chef/cache/veewee-gem
    - checkout ref 94608e0e16a54ac57008d0dad242768d8ddef05d branch HEAD
  * execute[build veewee gem] action run
    - execute gem build veewee.gemspec
  * gem_package[veewee] action install
    - install version 0.3.7 of package veewee
  * link[/opt/vagrant/bin/veewee] action create
    - create symlink at /opt/vagrant/bin/veewee to /opt/vagrant/embedded/bin/veewee
  * package[openjdk-6-jre-headless] action install (up to date)
  * file[/etc/profile.d/vagrant_path.sh] action create
    - create new file /etc/profile.d/vagrant_path.sh with content checksum e8ff03
        @@ -0,0 +1 @@
        +export PATH=$PATH:/opt/vagrant/bin
```