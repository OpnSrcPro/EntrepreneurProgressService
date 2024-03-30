Using Github actions to deploy Craft CMS websites
While I regularily use services like Buddy or DeployHQ to build and deploy Craft CMS websites, I wanted to experiment with Github Actions. Here is what I came up with and why.

Why Github Actions?
Services like Buddy or DeployHQ are really powerful. They allow you to setup complex build and deployment pipelines like atomic deployments using a nice GUI interface.

That being said, that power and ease of use come with a price tag which, in my opinion, can become quite high for basic projects. Most of the small to medium websites I work on do not necessarily need that level of power and complexity.

Since I generally use Github to host my repositories, being able to setup a basic deployment pipeline with a single YAML file for free seemed like an alternative worth investigating.

Deployment steps
My goal was to use Github Actions to create a basic build and deployment pipeline for Craft CMS websites. Here are the steps I needed:

Pull repository in container
Load SSH key (used by rsync)
Use specific version of Node
Install PHP dependencies with composer
Install Node dependencies
Build assets using NPM scripts
Deploy everything to the production server with rsync
Log into the production server and execute Craft commands
Security
Since this workflow needs to connect to servers using SSH, we have to create three Github Secrets in the repository we want to deploy:

SSH_HOST: the IP address of the server we need to connect to
SSH_USER: the SSH user
SSH_KEY: the SSH key. The recommended option here is to create a dedicated key pair for the repository rather than using your personal key. The public key will be stored on the server, while the private key will be stored in the Github Secret.
We will now be able to reference those secrets in our workflow using the following syntax: ${{ secrets.MYSECRET }}

The end result
After reading a couple of helpful articles and blogposts, here is the craftdeploy.yaml file I came up with.

name: Craft CMS deployments

on:
  push:
    branches: [master]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # Pull repository into the current pipeline
      - name: Pull repository
        uses: actions/checkout@v2

      # Setup container with private SSH Key (used by rsync)
      - name: Load private SSH key
        uses: webfactory/ssh-agent@v0.4.1
        with:
          ssh-private-key: ${{ secrets.SSH_KEY }}

      # Use a specific version of Node
      - name: Setup Node
        uses: actions/setup-node@v1
        with:
          node-version: "12.x"

      # Install PHP dependencies
      - name: Composer install
        run: composer install --no-interaction --no-progress --no-suggest --optimize-autoloader

      # Install NPM dependencies
      - name: NPM install
        run: npm ci

      # Build assets using NPM scripts
      - name: Build assets
        run: npm run build

      # RSYNC
      # - rsync [options] ~/localdir ssh_user@ssh_host:destination_directory
      # - exclude web/uploads is there to avoid deleting user uploaded files w/ --delete-after
      # - StrictHostKeyChecking=no will automatically add new host keys to the user known hosts files.
      - name: Deploy with rsync
        run: |
          rsync -azh --delete-after --exclude={"/web/uploads/","/node_modules/","/.git/","/.github/"} -e "ssh -o StrictHostKeyChecking=no" ./ ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}:~/

      # execute Craft commands on remote server
      - name: Execute SSH commmands on remote server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            chmod a+x craft
            php craft backup/db
            php craft migrate/all
            php craft project-config/apply
            php craft clear-caches/all
By creating a craftdeploy.yaml file in ./.github/workflows/ and by setting the push event to the master branch, we ensure that this workflow will run every time we push code to the master branch of our repository.

We use rsync to deploy files from the Github container to the production server. In the script above, we are copying files from the root of our local install to the root of a remote server we connect to via SSH.

Warning: rsync is a very powerful tool and can be quite destructive if you mess up your command. If you are new to it (and event if you aren't), consult the man page to see all available options and test it using the terminal. The --dry-run option, in particular, is very useful. It will not do anything and will print out the files that would have been transferred in your terminal.

I have been testing this script with several Craft projects for a couple of months and have not experienced any issue so far. The workflow is consistently taking between 1 and 3 minutes to run, which I find rather reasonable.

Granted, this is a very basic workflow and it can certainly be improved upon. Feel free to ping me me if you have ideas.

While I will probably continue to use Buddy and the likes for more complex needs, I am also quite happy with the convenient, cheap and reliable option Github Actions provides.

Need some help? Let's have a coffee together.
HOME
BLOG
WORK
CONTACT
© Webstoemp 2024
