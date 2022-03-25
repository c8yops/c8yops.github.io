# Upgrade of A1D environment to 10.11

!!!info Info: 
We make sure to remove the cores from the monitoring. 
!!!

1. Log into **management** node. Download new chef components from yum in your home folder.

2. Extract, rename the folder and copy into the c8y folder.

     ```bash
     tar -xvf [archive.tar]
     cd /root/cumulocity/
     mv /home/[user]/cumulocity-chef ./cumulocity-chef-release-pack-v[version]/
     ```

3. Copy from the old version the following folders and files

    - .chef

     ```bash
     cp -r ../cumulocity-chef/.chef/ ./
     ```

    - data bags

     ```bash
     cd data_bags/
     cp -r ../cumulocity-chef/data_bags/users_cumulocity/ ./
     ```

    - cumulocity-monitoring-agent

     ```bash
     cd cookbooks/
     rm -rf cumulocity-monitoring-agent/
     cp -r ../../cumulocity-chef/cookbooks/cumulocity-monitoring-agent/ ./
     ```

    - environments file

     ```bash
     cd environments/
     cp -r ../../cumulocity-chef/environments/[environemt-file] ./
     ```

4. Make the necessary changes in the environment file and any other changes that are present in the Worklog.

    !!!
    We check the connection to the environment with `knife node list`. 
    !!!

5. Remove one of the core nodes from the load balancers.

     - On **management** node

     ```bash
     knife node list
     knife node run_list remove [core] 'role[cumulocity-mn-active-core]'
     ```

     - On **load balancers**

     ```bash
     chef-client
     ```

6. Log into **Kubernetes master** node and check `cc-values.yml` file.

7. Go back to the **management** node, remove the symlink and direct the new one.

     ```bash
     rm cumulocity-chef
     ln -s cumulocity-chef-release-pack-v[version]/ cumulocity-chef
     ```

8. Upload the new cookbooks.

     ```bash
     knife cookbook upload --all
     knife environment from file environments/[environment-file]
     ```

9. Go to **Kubernetes master** node and run `chef-client`. Once finished, check `cc-values.yml` file and copy it into `/root/`.

     ```bash
     chef-client
     cd /etc/kubernetes/core
     cp cc_values.yml /root/values-[new].yml
     ```

10. Check the differences between the old and new file and make the necessary changes in the file.

     ```bash
     cd /root/
     diff values-[old].yml values-[new].yml
     vim values-[new].yml
     ```

11. Update the helm repo and do an upgrade using the new `cc-values.yml` file.

     ```bash
     helm repo update
     helm upgrade cumulocity-core stack/cumulocity-core -f /root/values-[new].yml --namespace=cumulocity-core --version [version]
     ```

12. Delete the pod with the core node and monitor until the core is back online.

     ```bash
     kubectl -n cumulocity-core get pods -o wide
     kubectl -n cumulocity-core delete pod c8ycore-sts-[X]
     ```

13. Monitor the DB index creation. Progress of the index migration is verified via checking Karaf's log file. The
process ends if no new entry appears in the log files: `MongoDB - Create index(db:{},
collection:{}, index:{}) ...`

     ```bash
     kc -n cumulocity-core logs -f c8ycore-sts-1 cumulocity-core | grep index
     ```

14. Check tenant health by logging into the **core** node and running the below command.

     ```bash
     curl -s http://127.0.0.1:8181/tenant/health | jq
     ```

15. Once everything has finished we bring back the core to the load balancers.

     - On **management** node

     ```bash
     knife node list
     knife node run_list add [core] 'role[cumulocity-mn-active-core]'
     ```

     - On **load balancers**

     ```bash
     chef-client
     ```

16. We do the same for the rest of the core nodes one-by-one.

     - On **management** node

     ```bash
     knife node list
     knife node run_list remove [core] 'role[cumulocity-mn-active-core]'
     ```

     - On **load balancers**

     ```bash
     chef-client
     ```

     - On **Kubernetes master**

     ```bash
     kubectl -n cumulocity-core get pods -o wide
     kubectl -n cumulocity-core delete pod c8ycore-sts-[X]
     ```

     - Once finished, we add the core back to the load balancer(Step 15) and repeat the above steps for the next core.

17. If not automatically upgraded, log back to the **core** and place the downloaded new microservices into the /2Images/ folder.

     ```bash
     cd /webapps/2Images/
     cp /home/[user]/ms-1011.0.12/*.zip ./
     ```

18. Check if lwm2m and ssl are installed on the agent node.