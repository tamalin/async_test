-----
# Ansible Async Testing
-----

## Prerequisites
-----

- Ansible >= 2.9.12 (tested with 2.9.27 & 2.10.7)
- Vagrant >= 2.2.0
- vagrant-libvirt >= 0.5.0 (tested with 0.5.3)
- KVM (i.e. qemu-kvm) newer is better. (tested with 4.2 on Ubuntu 20.04)
- Host with at least 16GB RAM and 250GB free disk space.


## Getting started
-----

1.  Copy the `settings.yml.example` file to `settings.yml` and customise as
    needed.

2.  Run `vagrant up` to bring up the test envs that are enabled to auto start,
    usually the most recent openSUSE Leap and Ubuntu LTS releases.

3.  Run `vagrant status` to see the full list of available test envs, and run
    `vagrant up <testenv>` to bring up additional testenvs

4.  VMs will be minimally provisioned, such as fixing known networking issues
    with the Vagrant box images.

5.  Run `ansible-galaxy collection install -r ansible/requirements.yml` to
    install Ansible requirements.

6.  Run `ansible-playbook ansible/all_in_one_async_test.yml` to run through
    all of the playbooks in a single run, or run each of the sub-playbooks
    in order (requires persistent fact caching to be configured), e.g.

    ```
    % ansible-playbook ansible/start_background_job.yml
    % ansible-playbook ansible/check_background_job.yml
    % ansible-playbook ansible/wait_for_background_job_completion.yml
    ```

### Logging in to a node
-----
If you run `vagrant ssh <testenv>` you can log in to the specified test env VM.

### Configuring Ansible persistent fact caching
-----
The `settings.yml.example` includes defaults that will ensure that the relevant
ansible.cfg settings are set correctly when generating both the `ansible.cfg`
and the `ansible/ansible.cfg` if they don't already exist.

If you want use a different persistent fact caching mechanism, you can do so by
updating the relevant settings in your `settings.yml` file, or by setting the
appropriate environment settings in these Ansible environment variables:

* ANSIBLE_CACHE_PLUGIN (`[defaults]fact_caching`)
* ANSIBLE_CACHE_PLUGIN_CONNECTION (`[defaults]fact_caching_connection`)
* ANSIBLE_CACHE_PLUGIN_PREFIX (`[defaults]fact_caching_prefix`)
* ANSIBLE_CACHE_PLUGIN_TIMEOUT (`[defaults]fact_caching_timeout`)

NOTE: Associated `ansible.cfg` setting in the `[defaults]` section shown in
parentheses after each environment variable.

See the `ansible-config list` output for details about the specific settings.

### Regenerating the ansible.cfg files
-----
If you change the ansible fact caching settings in your `settings.yml` file
after running a `vagrant` command, delete the `ansible.cfg` files and run
`vagrant status` to ensure they are regenerated.

### Updating boxes
-----
If `vagrant up` reports that updated boxes are available you can update the
associated Vagrant boxes in your local cache by running `vagrant box update`.
   
## Cleaning up
-----
Run `vagrant destroy -f` to cleanup all running test envs.