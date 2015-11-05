# Alfresco Vagrant Web Box
#
# -*- mode: ruby -*-
# vi: set ft=ruby :

# export COOKBOOKS_URL="https://artifacts.alfresco.com/nexus/service/local/repositories/snapshots/content/org/alfresco/devops/chef-alfresco/0.6.8-SNAPSHOT/chef-alfresco-0.6.8-20151104.130154-30.tar.gz"
# export STACK_TEMPLATE_URL=file://$PWD/stack-templates/enterprise-clustered.json

require 'json/merge_patch'

params = {}
boxAttributes = ""

params['downloadCmd'] = ENV['DOWNLOAD_CMD'] || "curl --silent"
params['workDir'] = ENV['WORK_DIR'] || "./.vagrant"
params['packerBin'] = ENV['PACKER_BIN'] || "/Users/mau/Documents/opt/packer-0.7.5/packer"
params['packerOpts'] = ENV['PACKER_OPTS'] || ''

params['boxUrl'] = ENV['BOX_URL'] || "http://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_centos-7.1_chef-provisionerless.box"
params['boxName'] = ENV['BOX_NAME'] || "opscode-centos-7.1"

params['cookbooksUrl'] = ENV['COOKBOOKS_URL'] || "https://artifacts.alfresco.com/nexus/service/local/repositories/releases/content/org/alfresco/devops/chef-alfresco/0.6.7/chef-alfresco-0.6.7.tar.gz"
params['dataBagsUrl'] = ENV['DATABAGS_URL'] || ''

params['stackTemplateUrl'] = ENV['STACK_TEMPLATE_URL'] || "file://#{ENV['PWD']}/stack-templates/community-allinone.json"

def printVars(params)
  print "#START - Printing out Vagrant environment variables:\n"
  params.each do |paramName,paramValue|
    print "#{paramName}: '"+paramValue+"'\n"
  end
  print "#END - Printing out Vagrant environment variables:\n"
end

def reloadChefItems(nodes,params)
  # Download and uncompress Chef cookbooks (in a Berkshelf package format)
  if params['cookbooksUrl'] and params['cookbooksUrl'].length != 0
    `#{params['downloadCmd']} #{params['cookbooksUrl']} > #{params['workDir']}/cookbooks.tar.gz`
    print "Downloaded #{params['cookbooksUrl']}\n"
    `rm -rf #{params['workDir']}/cookbooks; tar xzf #{params['workDir']}/cookbooks.tar.gz -C #{params['workDir']}`
    print "Unpacked #{params['workDir']}/cookbooks.tar.gz into #{params['workDir']}\n"
  end

  # Download and uncompress Chef databags
  if params['dataBagsUrl'] and params['dataBagsUrl'].length != 0
    `#{params['downloadCmd']} #{params['dataBagsUrl']} > #{params['workDir']}/databags.tar.gz`
    print "Downloaded #{params['dataBagsUrl']}\n"
    `rm -rf #{params['workDir']}/databags; tar xzf #{params['workDir']}/databags.tar.gz -C #{params['workDir']}`
    print "Unpacked #{params['workDir']}/databags.tar.gz into #{params['workDir']}\n"
  end

  # Download node URL
  nodes.each do |chefNodeName,chefNode|
    print "Processing node '#{chefNodeName}'\n"
    `#{params['downloadCmd']} #{chefNode['instance-template']} > #{params['workDir']}/attributes-#{chefNodeName}.json.original`
    print "Downloaded #{chefNode['instance-template']} into #{params['workDir']}/attributes-#{chefNodeName}.json.original\n"

    # For debugging purposes
    # print "localVars: #{chefNode['localVars'].to_json}\n"

    # Patch node URL with localVars
    mergedAttributes = JSON.parse(JSON.merge(File.read("#{params['workDir']}/attributes-#{chefNodeName}.json.original"), chefNode['localVars'].to_json))

    if ENV['NEXUS_USERNAME'] and ENV['NEXUS_PASSWORD']
      mergedAttributes['artifact-deployer'] = {}
      mergedAttributes['artifact-deployer']['maven'] = {}
      mergedAttributes['artifact-deployer']['maven']['repositories'] = {}
      mergedAttributes['artifact-deployer']['maven']['repositories']['private'] = {}

      mergedAttributes['artifact-deployer']['maven']['repositories']['private']['url'] = "https://artifacts.alfresco.com/nexus/content/groups/private"
      mergedAttributes['artifact-deployer']['maven']['repositories']['private']['username'] = ENV['NEXUS_USERNAME']
      mergedAttributes['artifact-deployer']['maven']['repositories']['private']['password'] = ENV['NEXUS_PASSWORD']
    end

    attributeFile = File.open("#{params['workDir']}/attributes-#{chefNodeName}.json", 'w')
    attributeFile.write(mergedAttributes.to_json)
    attributeFile.close()

    print "Merged #{params['workDir']}/attributes-#{chefNodeName}.json.original and #{params['workDir']}/localVars.json into #{params['workDir']}/attributes-#{chefNodeName}.json\n"
  end
end

print "Running Vagrant #{ARGV[0]}\n"

# Make sure the work directory exists
`mkdir -p #{params['workDir']}/packer`
`mkdir -p #{params['workDir']}/alf_data`

# Download nodes URL
`#{params['downloadCmd']} #{params['stackTemplateUrl']} > #{params['workDir']}/nodes.json`
print "Downloaded #{params['stackTemplateUrl']} into #{params['workDir']}/nodes.json\n"

# Load nodes JSON
nodes = JSON.parse(File.read("#{params['workDir']}/nodes.json"))

if ['packer'].include? ARGV[0]
  reloadChefItems(nodes, params)

  packerDefs = {}

  # Define Packer suites
  nodes.each do |chefNodeName,chefNode|
    builderUrls = chefNode['packer']['builders']
    provisionerUrls = chefNode['packer']['provisioners']

    # Collect parsed builders and provisioners here
    builders = "["
    provisioners = "["

    # Download provisioners JSON files
    provisionerUrls.each do |provisionerName,provisionerUrl|
      `#{params['downloadCmd']} #{provisionerUrl} > #{params['workDir']}/packer/#{provisionerName}-provisioner.json`
      print "Downloaded #{provisionerUrl} into #{params['workDir']}/packer/#{provisionerName}-provisioner.json\n"
      provisioner = File.read("#{params['workDir']}/packer/#{provisionerName}-provisioner.json")

      # Inject Chef attributes JSON into the chef-solo provisioner
      # Please note that the JSON have been already patched with localVars
      # by reloadChefItems function
      if provisionerName == 'chef-solo'
        provisionerJson = JSON.parse(provisioner)
        nodeUrl = "#{params['workDir']}/attributes-#{chefNodeName}.json"
        nodeUrlContent = File.read("#{params['workDir']}/attributes-#{chefNodeName}.json.original")
        provisionerJson['json'] = JSON.parse(nodeUrlContent)
        provisionerFile = File.open("#{params['workDir']}/packer/#{provisionerName}-provisioner.json", 'w')
        provisionerFile.write(provisionerJson.to_json)
        provisionerFile.close()
        provisioner = File.read("#{params['workDir']}/packer/#{provisionerName}-provisioner.json")
      end

      provisioners += provisioner + ","
    end

    # Download builders JSON files
    builderUrls.each do |builderName,builderUrl|
      `#{params['downloadCmd']} #{builderUrl} > #{params['workDir']}/packer/#{builderName}-builder.json`
      print "Downloaded #{builderUrl} into #{params['workDir']}/packer/#{builderName}-builder.json\n"
      builders += File.read("#{params['workDir']}/packer/#{builderName}-builder.json") + ","
    end

    builders = builders[0..-2]
    builders += "]"
    provisioners = provisioners[0..-2]
    provisioners += "]"

    # Compose Packer JSON
    packerDefs[chefNodeName] = "{\"builders\":#{builders},\"provisioners\":#{provisioners}}"
  end

  # Summarise Packer suites and ask for confirmation before running it
  print "Running the following Packer templates:\n"
  packerDefs.each do |packerDefName,packerDef|
    print "- #{packerDefName}-packer.json\n"
  end
  # print "Are you sure you want to continue? (yes/no)\n"
  # STDOUT.flush
  # answer = gets.chomp
  # unless answer == 'yes'
  #   exit 0
  # else
    # Run Packer suites
    packerDefs.each do |packerDefName,packerDef|
      packerFile = File.open("#{params['workDir']}/packer/#{packerDefName}-packer.json", 'w')
      packerFile.write(packerDef)
      packerFile.close()
      `cd #{params['workDir']}/packer; #{params['packerBin']} build #{packerDefName}-packer.json #{params['packerOpts']}`
    end
  # end
else
  if ['up','provision'].include? ARGV[0]
    reloadChefItems(nodes, params)
  end

  Vagrant.configure("2") do |config|
    # TODO - make pretty printing work
    #
    # boxAttributesContent = JSON.pretty_generate(reloadChefItems(params))
    nodes.each do |chefNodeName,chefNode|
      config.vm.define chefNodeName do |machineConfig|
        boxAttributesContent = File.read("#{params['workDir']}/attributes-#{chefNodeName}.json")
        # boxAttributes = JSON.pretty_generate(JSON.parse(boxAttributesContent))
        boxAttributes = JSON.parse(boxAttributesContent)
        # For debugging purposes
        # print("boxAttributes:\n#{boxAttributes}\n")

        boxIp = boxAttributes["ip"]
        boxHostname = boxAttributes["hostname"] || boxAttributes["name"]
        boxRunList = boxAttributes["run_list"]

        # Box env configuration
        if boxIp
          config.vm.network :private_network, ip:  boxIp
        end
        machineConfig.vm.hostname = boxHostname
        machineConfig.vm.provider :virtualbox do |vb,override|
          override.vm.box_url = params['vagrantBoxUrl']
          override.vm.box = params['boxName']
          vb.customize ["modifyvm", :id, "--memory", chefNode['memory']]
          vb.customize ["modifyvm", :id, "--cpus", chefNode['cpus']]
        end

        # Use the latest Chef version
        # config.omnibus.chef_version = "12.5.1"

        # Chef run configuration
        machineConfig.vm.provision :chef_solo do |chef|
          chef.cookbooks_path = "#{params['workDir']}/cookbooks"
          chef.data_bags_path = "#{params['workDir']}/data_bags"

          boxRunList.each do |recipe|
            chef.add_recipe "recipe[#{recipe}]"
          end

          chef.json = boxAttributes
        end
      end
    end
  end
end
