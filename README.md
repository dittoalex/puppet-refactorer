puppet-refactorer
=================

Interactively refactor puppet manifests of live nodes in a production environment

Refactoring a live puppet node when migrating it to a new puppet repo

!!!
Modify http://www.devco.net/code/parselocalconfig.rb to work with your puppet version

        [user@organization-prod-role107 ~]$ diff -u parselocalconfig.rb*
        --- parselocalconfig.rb 2014-06-03 14:46:33.081149506 -0500
        +++ parselocalconfig.rb.default 2010-09-14 12:31:27.000000000 -0500
        @@ -61,7 +61,7 @@
         if Puppet.version =~ /^([0-9]+[.][0-9]+)[.][0-9]+$/
           version = $1

        -  unless ["0.25", "0.24", "2.6", "2.7" ].include?(version)
        +  unless ["0.25", "0.24", "2.6"].include?(version)
             puts("Don't know how to print catalogs for verion #{Puppet.version} only 0.24, 0.25 and 2.6 is supported")
             exit 1
           end
!!!

* Identifying which resources to transfer

An example is organization-prod-role107 whose master is old_puppetmaster and we want to move to new_puppetmaster.  It has many used classes, some of which have been superseded or have 'organic' resources.  Its used classes are:  

        Classes included on this node:
                organization::type::prod-role-partner
                ibm_puppet_setup
                international_business_machines
                settings
                default
                ibm_puppet_setup::stages
                organization::type::prod-role-partner
                organization
                timezone
                ibm_users
                ibm_users::users
                ibm_users::virtual
                ibm_users::daemons
                ibm_sudoers
                organization::awsautoconfig
                organization::env::prod
                organization::component::ganglia
                organization::component::nrpe
                organization::component::organization_user
                organization::component::java
                organization::component::partner_roleer_user
                organization::component::partner_user
                organization::component::nutch
                ibm_puppet_setup
                international_business_machines
                international_business_machines::params
                international_business_machines::dns
                international_business_machines::post_provision
                international_business_machines::repos
                international_business_machines::removal
                international_business_machines::installation
                international_business_machines::latest
                cpan
                ntp
                ntp::params
                ibm_scripts
                ibm_scripts::install
                java
                ibm_aws
                international_business_machines::virtuals
                international_business_machines::rc_local
                international_business_machines::firewall
                international_business_machines::services

Parse the local puppet catalog and reformat it so it can be more easily read

        $ sudo ./parselocalconfig.rb --nc /var/lib/puppet/client_yaml/catalog/$(hostname --fqdn).yaml  | tail -n +2 | grep . | paste -d ' ' - - | sort -n  > puppet.local_catalog.parsed.resources

Looking through puppet.local_catalog.parsed.resources may be a bit overwhelming at first:

        Augeas { password_auth: }               defined in /etc/puppet/modules/common/ibm_users/manifests/init.pp:16
        Class { organization::Component::Java: }           defined in /etc/puppet/modules/organization/manifests/type/prod-role-partner.pp:9
        Exec { cpan_install_Digest-MD5: }               defined in /etc/puppet/modules/organization/manifests/utils/cpan_install.pp:17
        File { /var/log/ibm: }          defined in /etc/puppet/modules/organization/manifests/init.pp:102
        Firewall { 999 deny all other requests: }               defined in /etc/puppet/modules/common/international_business_machines/manifests/firewall.pp:105
        Group { group: }                defined in /etc/puppet/modules/common/ibm_users/manifests/virtual.pp:26
        Host { organization-prod-role107.REALM.COM: }          defined in /etc/puppet/modules/common/international_business_machines/manifests/dns.pp:60
        Package { ack: }                defined in /etc/puppet/modules/common/international_business_machines/manifests/installation.pp:9
        Resources { firewall: }                 defined in /etc/puppet/modules/common/international_business_machines/manifests/firewall.pp:40
        Resources { user: }             defined in /etc/puppet/modules/common/ibm_users/manifests/init.pp:6
        Service { autofs: }             defined in /etc/puppet/modules/common/international_business_machines/manifests/services.pp:30
        organization::Utils::Cpan_install { Class::Std::Utils: }           defined in /etc/puppet/modules/organization/manifests/init.pp:316
        ibm_sudoers::Create { ops: }            defined in /etc/puppet/modules/common/ibm_users/manifests/init.pp:44
        ibm_users::Virtual::Localuser { user: }             defined in /etc/puppet/modules/common/ibm_users/manifests/users.pp:738
        User { group: }                 defined in /etc/puppet/modules/common/ibm_users/manifests/virtual.pp:20

!!
These functions can be sourced from repomanium/puppet/refactor_manifests.sh.lib
!!

For a high-level overview, check the resources by module

        $resources_by_module puppet.local_catalog.parsed.resources
        ## Resources by module
         331 organization
         454 common

Check the resources by file $resources_by_file

        ## Resources by file
           1 /etc/puppet/modules/common/cpan/manifests/init.pp
           1 /etc/puppet/modules/common/ibm_scripts/manifests/init.pp
          <snip>
          25 /etc/puppet/modules/organization/manifests/utils/cpan_install.pp
          28 /etc/puppet/modules/common/ibm_scripts/manifests/install.pp
          33 /etc/puppet/modules/organization/manifests/component/partner_roleer_user.pp
          46 /etc/puppet/modules/common/ibm_users/manifests/users.pp
          53 /etc/puppet/modules/organization/manifests/component/organization_user.pp
          80 /etc/puppet/modules/common/ibm_users/manifests/virtual.pp
         108 /etc/puppet/modules/organization/manifests/init.pp
         200 /etc/puppet/modules/common/international_business_machines/manifests/installation.pp

The resources by type will allow a quick insight as to how things are broken down and will provide more insight into defined types

         ## 20:31:04 0 user@localhost:/e$resources_by_type puppet.local_catalog.parsed.resources
        ## Resources by type
           1 Cron
           1 Host
           1 ibm_sudoers::Create
           1 international_business_machines::Dns::Rename
           1 international_business_machines::Removal::Pkgremove
           1 international_business_machines::Services::Svcenable
           2 Resources
           2 international_business_machines::Post_provision::Wipeout_annoying_files
           3 Augeas
           3 organization::Utils::Mount_ebs
           3 organization::Utils::Route53_dns_alias
           5 organization::Utils::Set_ec2_tag
           6 organization::Utils::S3_tarball_install
           9 international_business_machines::Services::Svcdisable
          12 Firewall
          16 Service
          25 organization::Utils::Cpan_install
          32 Class
          40 ibm_users::Virtual::Localuser
          44 User
          51 Group
          65 Exec
          72 international_business_machines::Installation::Pkgdeclare
          72 international_business_machines::Installation::Pkginstall
         150 Package
         167 File

Let's get into modifying the puppet.local_catalog.parsed.resources file to remove unnecssary resources.

Open a terminal and use resource_by_file_loop() to keep a track of the most resources by file.

Have vim only display the resources by that file:
        :g#international_business_machines/manifests/installation.pp
                Package { ack: }                defined in /etc/pup
        pet/modules/common/international_business_machines/manifests/instal
        lation.pp:9
                Package { aspell: }             defined in /etc/pup
        pet/modules/common/international_business_machines/manifests/instal
        lation.pp:9
        <snip>
        mon/international_business_machines/manifests/instal
        lation.pp:9
        -- More --

We know ibm_users/manifests/init.pp has been superseded by a new module on new_puppetmaster and that nothing from ibm_users/manifests/init.pp is needed, so we can delete all the lines with:

         :g#international_business_machines/manifests/installation.pp#d
        200 fewer lines
        Press ENTER or type command to continue

Write the changes to file
        <.parsed.resources" 585L, 66887C written 490,2-9       83%>

The resource_by_file_loop() will then update to show the sorted list without the items there

        ## Resources by file
           1 /etc/puppet/modules/common/cpan/manifests/init.pp
           1 /etc/puppet/modules/common/ibm_scripts/manifests/init.pp
          <snip>
          25 /etc/puppet/modules/organization/manifests/utils/cpan_install.pp
          28 /etc/puppet/modules/common/ibm_scripts/manifests/install.pp
          33 /etc/puppet/modules/organization/manifests/component/partner_roleer_user.pp
          46 /etc/puppet/modules/common/ibm_users/manifests/users.pp
          53 /etc/puppet/modules/organization/manifests/component/organization_user.pp
          80 /etc/puppet/modules/common/ibm_users/manifests/virtual.pp
         108 /etc/puppet/modules/organization/manifests/init.pp

You now have 200 fewer resources to wrangle.  Repeat this for the other listed files while following these resource classification rules:
  1.)  If that resource has been superseded by another base module (ibm_base), remove it.
  2.)  If the resource has been moved to another manifest, reference the new location.  You can search /etc/puppet/modules on new_puppetmaster to verify this:

        [user@xibmprodnew_puppetmaster modules]$ sudo -E ack --all --ignore-case nagios-plugins-nrpe
        ibm/component/manifests/class/nrpe.pp
        3:  #$nrpePackages = [ "nrpe","nagios-plugins-nrpe", "nagios-plugins-all","perl-Filesys-Df","percona-nagios-plugins", "nagios-common","nsca" ]
        4:  #$nrpePackages = [ "nrpe","nagios-plugins-nrpe", "nagios-plugins-all", "percona-nagios-plugins" ]

        ibm/ibm_base/manifests/packages.pp
        146:       "nagios-plugins-nrpe",
        
     Note that resource titles may be misleading as variables passed through a defined type or custom resource may alter the resource name on the locally compiled catalog.  One example is the resource title install_nutch_1_8 not being statically defined in a manifest, as it is called by line 184 in nutch.pp
     
[user@organization-prod-role107 ~]$ fgrep nutch_1_8 puppet.local_catalog.parsed.resources
        Exec { install_nutch_1_8: }             defined in /etc/puppet/modules/organization/manifests/utils/s3_tarball_install.pp:50
        organization::Utils::S3_tarball_install { nutch_1_8: }   defined in /etc/puppet/modules/organization/manifests/component/nutch.pp:142

./organization/manifests/component/nutch.pp:179:   file { '/usr/local/apache-nutch-1.8-bin/runtime/local/urls-tmp':
./organization/manifests/component/nutch.pp:180:           ensure          => directory,
./organization/manifests/component/nutch.pp:181:           mode            => 777,
./organization/manifests/component/nutch.pp:182:           group           => organization,
./organization/manifests/component/nutch.pp:183:           force           => true,
./organization/manifests/component/nutch.pp:184:           require         => organization::Utils::S3_tarball_install [ 'nutch_1_8' ],
     
  3.)  If the resource does not exist in the new repo, create it in an appropriate manifest.
  
The results for this example are the following resources to be part of the new_puppetmaster manifest:
Modules it will need:
nrpe
 nutch (old_puppetmaster installs 1.8;  new_puppetmaster's nutch component is 1.4)
java
ganglia (exists on new_puppetmaster)

Resources that it will need:
Group { organizationeng: } 
File { /etc/sudoers.d/env-prod: }               defined in /exampletc/puppet/modules/organization/manifests/env/prod.pp:31
File { /root/.Anws/config: }             defined in /etc/puppet/modules/organization/manifests/env/prod.pp:19

Manifests that should be migrated:
  2 /etc/puppet/modules/organization/manifests/component/partner_user.pp
     10 /etc/Puppetppet/modules/organization/manifests/type/prod-role-partner.pp
     33 /endtc/puppet/modules/organization/manifests/component/partner_roleer_user.pp
     53 /etc/puppet/modules/organization/manifests/component/organization_user.pp 
 
* Refactoring manifest architecture

It may be the case that the new puppet repository does things very differently.  If  you choose to copy the manifests to new_puppetmaster rather than rewrite them from the bottom up, you should identify unnecessary directives and make the blanket change.  Eg:  new_puppetmaster does not use stages.  

-class organization::type::prod-nutch-eph ( $env = $::foreman_env, $stage = "verticalSetup" ) {
+class organization::type::prod-nutch-eph {

* Re-assembling resources into a manifest

Work from the bottom up in terms of nested manifests:  first port the partner_roleer_user.pp manifest, because it includes no other components.  Then the type/prod-role-partner.  (?  note:  not checked)
