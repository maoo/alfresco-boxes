# Alfresco SPK Vagrantfile
#
# -*- mode: ruby -*-
# vi: set ft=ruby :
require_relative 'scripts/provisioning-libs'

params = getEnvParams()
initWorkDir(params['workDir'])
nodes = getStackTemplateNodes(params['downloadCmd'], params['workDir'], params['stackTemplateUrl'])

vagrantCommand = ARGV[0]
print "Running Vagrant #{vagrantCommand}\n"

if ['packer'].include? vagrantCommand
  downloadChefItems(nodes, params['workDir'], params['downloadCmd'], params['cookbooksUrl'], params['dataBagsUrl'])
  packerDefs = getPackerDefinitions(nodes)
  runPackerDefinitions(nodes, params['workDir'], params['packerBin'], params['packerOpts'])
else
  if ['up','provision'].include? vagrantCommand
    downloadChefItems(nodes, params['workDir'], params['downloadCmd'], params['cookbooksUrl'], params['dataBagsUrl'])
  end

  Vagrant.configure("2") do |config|
    nodes.each do |chefNodeName,chefNode|
      config.vm.define chefNodeName do |machineConfig|
        boxAttributes = getNodeAttributes(params['workDir'], chefNodeName)

        boxIp = boxAttributes["ip"]
        boxHostname = boxAttributes["hostname"] || boxAttributes["name"]
        boxRunList = boxAttributes["run_list"]

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