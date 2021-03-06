#Ansible playbooks

This reposiory contains playbooks to deploy the infrastructure at National Genomics
Infrastructure.

Please refer to [ansible documentation](http://docs.ansible.com/) for more details
about Ansible concepts and how it works.

Also, take a look at the ansible [best practices](http://docs.ansible.com/playbooks_best_practices.html).

The repository is structured following this structure
```
production                # inventory file for production servers
stage                     # inventory file for stage environment

group_vars/
   group1                 # here we assign variables to particular groups
   group2                 # ""
host_vars/
   hostname1              # if systems need specific variables, put them here
   hostname2              # ""

site.yml                  # master playbook
webservers.yml            # playbook for webserver tier
dbservers.yml             # playbook for dbserver tier

roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/           # ""
    fooapp/               # ""
```

In order to deploy any of the playbooks, clone this repository to a machine that
has `ssh` connection to all the servers you want to deploy. Usually your local machine
should be enough.

__NOTE__: The following assume you have ansible installed in your machine, if not:

```bash
$> sudo pip install ansible
```

### Create a testing environment
You don't want to mess around the production servers until you are sure that everything
works, don't you? Okay that's easy to solve: just create a virtual mini-cluster
using vagrant and configure it [like this](http://hakunin.com/six-ansible-practices#build-a-convenient-local-playground)

# Ansible and 2-factor authentication
Ansible does not play nice with 2-factor authentication by the moment, so in order
to run a playbook on a server 2-factor authentication enabled you have 2 options:

1. Clone this deployment repository into the server and run the playbook locally
using `--connection=local`

2. _Slightly more complicated_: Tunnel the connection through an already opened
connection using [ControlMaster](http://www.anchor.com.au/blog/2010/02/ssh-controlmaster-the-good-the-bad-the-ugly/)
ssh option:

    * Create or edit your `~/.ssh/config` file and add the following to create a master connection
    per server:

    ```
    Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm_socket/%r@%h:%p
    ```

    * Create or edit your `~/.ansible.cfg` configuration file to use that `ControlMaster` connection:

    ```
    [ssh_connection]
    ssh_args = -o ControlMaster=auto -o ControlPath=<your_home_directory>/.ssh/cm_socket/%r@%h:%p
    ```

    * Open a regular connection to the server, like `ssh user@server`, enter the vault token and your password.
    **Keep the connection opened.**

    * On a separate terminal, run the playbook as usual.

## Deploying a new NAS
In order to deploy a new NAS, you just need to execute the `nas` playbook. Some
**important** considerations before doing that are:

* You need to be able to ssh paswordless into the NAS
* You need permission to `sudo` to the production user (which you can ask to some NGI member)

Once this is clear:

```bash
$> ansible-playbook nas.yml -u <your_username> -i production --ask-vault-pass
```

 There are 2 variables defined in the role vars file that you _may_ want to replace:
`dest_server` and `dest_data`. These variables define the processing server where the data is going to be synched and
the destination directory. You can
override it by adding `--extra-vars "dest_server=<value> dest_data=value"`. The
default values for these variables are the ones needed **for a NAS connected to an
XTen machine**.

This playbook will:

1. Install `miniconda` and create a virtual environment called `master` with all needed dependencies
2. Create `config` directory and put there all configuration files
3. Create `log` directory
4. Install several repositories all needed code repositories
5. Configure cronjobs
6. Start `supervisord`, which will start all necessary services
7. Copy irods credentials and configure iCommands (credentials are encrypted using `ansible-vault`, ask... someone for the password)


## Deploying a new processing server

To deploy a new processing server, execute the `processing` playbook. Same **important**
considerations from the nas need to be taken when deploying a processing server.

```bash
$> ansible-playbook processing.yml -u <your_username> -i production --ask-vault-pass
```

This playbook will:

* Create a public ssh key if not present
* Install miniconda and create a virtual environment called master
* Create necessary directories
* Install CASAVA 2.5 under ~/opt
* Copy necessary files
