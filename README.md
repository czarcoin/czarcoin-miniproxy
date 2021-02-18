# czarcoin-miniproxy

This app is a mini Express app "gateway" to the Storj library built using libstorj.

Functionalities:
  * Listing current bridge user's buckets
  * Adding and removing buckets
  * Uploading, downloading, and removing files (image/text only) from a bucket
  
## Sections
* [1. Connecting to a Bridge Server with Storj-Integration](#1-connecting-to-a-bridge-server-with-storj-integration)
* [2. Register a Bridge User](#2-register-a-bridge-user)
* [3. Activate Your Bridge User](#3-activate-your-bridge-user)
* [4. Installing and Running the Express App](#4-installing-and-running-the-express-app)
* [Useful Docker Commands](#useful-docker-commands)
* [Development Process](#development-process)
* [Connecting to a Bridge Server with Storj-SDK](#connecting-to-a-bridge-server-with-storj-sdk)
* [Troubleshooting Bridge Access with Storj-SDK](#troubleshooting-storj-sdk)
* [Work in Progress - Building from Scratch](#work-in-progress-building-from-scratch)

## Instructions

### 1. Connecting to a Bridge Server with [Storj-Integration](https://github.com/Storj/integration)
First you need to git clone both this repo _and_ [Storj-Integration](https://github.com/Storj/integration) into a `projects/` folder.

Then `cd` into your storj-integration directory and use the following command to build the docker image:
```
docker build -t storj-integration .
```
Then, use the following command to create a container with exposed ports to simulate farmers and enable uploads/downloads:
```
docker create -p 6382:6382 -p 3000:3000 -p 9000:9000 -p 9001:9001 -p 9002:9002 -p 9003:9003 -p 9004:9004 -p 9005:9005 -p 9006:9006 -p 9007:9007 -p 9008:9008 -p 9009:9009 -p 9010:9010 -p 9011:9011 -p 9012:9012 -p 9013:9013 -p 9014:9014 -p 9015:9015 -p 9016:9016 -t -i storj-integration bash
```
You will then be given a 32-bit hex number, which is your container id.
Then you can then start and attach to the container with:
```bash
docker start -ai <container-id>
```
When it prompts with `root@<container-id>:~#`, the last step is to run the `start_everything.sh` script:
```bash
./scripts/start_everything.sh
```
As long as you want to connect to the bridge server, you need to keep this terminal window running.
You can then use pm2 commands to view logs etc.

### 2. Register a Bridge User
Every time you start a new integration instance, in a separate window you're going to need to register a new bridge user with the CLI. This is because bridge users are stored in a Mongo collection inside each integration container.
You can register a new user with the command:
```
storj -u http://localhost:6382 register
```
This specifically registers a user with the local integration instance that you’re running using port 6382. You will then be prompted for your bridge username (email) and password. The Storj CLI will generate a secret key for you which it will use to encrypt your files, but first it will ask you what encryption strength to use (128, 160, 192, 224, or 256, with 256-bit being the strongest and recommended). Once you input your secret key strength, you will be given your encryption key, a 24-word mnemonic.

In yet another window, `cd` into your `storj-miniproxy/` directory and create a `.env` file.
You should then write the following values into this file to protect these variables.
```
BRIDGE_URL=’http://localhost:6382’
BRIDGE_EMAIL=’<same bridge email registered with the CLI>’’
BRIDGE_PASS=’<same password registered with the CLI>’
ENCRYPT_KEY=’<this is the 24-word mnemonic secret key that the CLI created for you>’
```
Keeping these bridge credentials in a dotenv file allows the storj-miniproxy app to decrypt and interact with your bridge user's files and buckets.

### 3. Activate Your Bridge User
Now you'll need to register your new CLI user's credentials inside the mongo user collection in your integration instance.
First use the docker shell to get into a mongo shell. Next to `root@<container-id>:~#`, type:
```bash
mongo
```
Then connect to the `storj-sandbox` database and look for users:
```bash
db = connect('storj-sandbox')
db.users.find({})
```
If you find that the bridge user account that you want to use has `"activated"` set to `"false"` in the user object, then you can use the following to activate:
```bash
db.users.findOneAndUpdate({_id:<your user email string>}, {$set:{activated: true, activator: null}});
```
You can check if your bridge user credentials are working using the `list-buckets` command:
```bash
storj -u http://localhost:6382 list-buckets
```
Note: Port 6382 is used with the integration bridge. If you're using the SDK, you should use `http://<bridge address>`.
To check your current bridge username, password, and encryption key, you can also use the command:
```bash
storj -u http://localhost:6382 export-keys
```

### 4. Installing and Running the Express App

After activating your bridge user credentials inside the integration container's mongo collection, `cd` back to the storj-miniproxy repo and use `npm install` then `npm start`.

[3/05/18 Note] If you already have [libstorj](https://github.com/Storj/libstorj) installed on your computer, you may receive `gyp ERR! build error`s when attempting to `npm install`. If this is the case, you need to go to your `libstorj` directory and `sudo make uninstall`. Then go back to the `storj-miniproxy` directory and try to `npm install` and `npm start` again.
(This problem occurs because storj-miniproxy uses node-libstorj, which conflicts with the current release of libstorj.)

Then you can check `http://localhost:7000` to see if the app's index page is there. (The reason for not using `http://localhost:3000` is because Docker runs on that port, which you will need to run your Storj test server.)
You can then use your browser to view your buckets, add buckets, upload/download, then delete buckets/files.

## Useful Docker Commands

If at any point you stop your storj-integration instance and want to return to it, you can get its container id as follows:
```bash
docker ps -a|grep storj-integration
```

To see what docker containers are running:
```bash
docker ps
```

To stop a specific container from running:
```bash
docker stop <container id>
```

To stop all running docker containers:
```bash
docker stop $(docker ps -q)
```

## Development Process

#### Dependencies Used
  * [node-libstorj](https://github.com/Storj/node-libstorj)
  * [dotenv](https://github.com/motdotla/dotenv)
  * [multer](https://github.com/expressjs/multer)

#### Current Goals
 - Rewrite routes without the redundant endpoint names.
 - Fix Travis build error.

#### Issues
- Files can't be uploaded directly from the clientside to the Storj library (Reed Solomon requires entire file, not streamed parts), so I'm using multer to save uploaded files to the local server (in `uploads/`), and then running the bridge method `storeFile`.
- Running into network error when uploading files using the sdk. Successfully retrieves frame id and creates frame, which is good, but when it starts `Pushing frame for shard index 0...`, begins to receive this error:
```
{"message": "fn[push_frame] - JSON Response: { "error": "getaddrinfo ENOTFOUND landlord landlord:8081" }", "level": 4, "timestamp": 1510079825998}

Error: Unable to receive storage offer
    at Error (native)
```
- NB: When attempting to use the `bucketList` route to list buckets, I ran into the following error:
```
GET /bucketList - - ms - -
Error: Not authorized
    at Error (native)
```
This was because I had the incorrect `BRIDGE_EMAIL`, `BRIDGE_PASS`, and `ENCRYPT_KEY` in my .env file.

### Connecting to a Bridge Server with [Storj-SDK](https://github.com/Storj/storj-sdk)

See [Storj-SDK](https://github.com/Storj/storj-sdk) README for setup.
Inside your storj-sdk repo, you can check what containers are running with this command:
```bash
docker-compose ps
```
To run the bridge:
```bash
docker-compose up -d
```
To set the bridge address:
```bash
source ./scripts/setbr
```
To set host entries:

(Note that as of 11/3/17, this script will do its job but automatically close your shell when it's finished.)
```bash
source ./scripts/set_host_entries.sh
```
You can confirm where your local bridge is set with:
```bash
source ./scripts/get_local_bridge.sh
```

You'll then be given the address to where the bridge is set. The `BRIDGE_URL` variable needs to be set to this address in your .env file.
In your `~/.storj` directory (for OSX) there should be some IP.json files. Make sure that the credentials in that file match your local bridge's IP address as well. You may have to rename your .json file with the correct IP address.

Now, when you use the command `storj export-keys`, the resulting email, password, and encryption key need to be saved to your `.env` file respectively as `BRIDGE_EMAIL`, `BRIDGE_PASS`, and `ENCRYPT_KEY`.

### Troubleshooting Bridge Access with Storj-SDK
Once inside the storj-sdk directory...
To check your hosts:
```bash
cat /etc/hosts
```
Use `mongo` to check which port that mongo is using. You should get back:
```
MongoDB shell version vx.x.x
connecting to: mongodb://127.0.0.x.yourPortNumber
```
To enter the mongo shell:
```
mongo db:yourPortNumber
```
You'll want to see what databases are available with `show dbs`, then `use storj-sandbox`.
You can view tables with `show tables`.
In order to find your activated bridge user, you can use:
```
db.users.find()
```
Then find the entry where `"activated" : true`.

In your `~/.storj` directory (for OSX) there should be a list of IP.json files. Make sure that the credentials in that file match the IP given by the `setbr` script inside `storj-sdk/`.
You may have to rename your .json file with the correct IP address.

Finally, make sure that you are connected to the storj-local VPN! See the storj-sdk README for specific setup.

## Work in Progress - Building from Scratch

2/17/18 - Note that if you first install libstorj then try to install node-libstorj, this will cause node-libstorj will break. So for this project, you will just need node-libstorj which you should install like this:
```
npm install github:storj/node-libstorj --save
```
The dotenv module can be installed the normal way with `npm install dotenv --save`.

#### 1. Create a Local Storj Development Network
First you will need to git clone the Storj Integration repo with the following command:
```
git clone git@github.com:Storj/integration.git 
```
Then `cd` into your new storj-integration repo and use this command to build the docker image:
```
docker build -t storj-integration .
```
Then, use the following command to create a container with exposed ports to simulate farmers and enable uploads/downloads:
```
docker create -p 6382:6382 -p 3000:3000 -p 9000:9000 -p 9001:9001 -p 9002:9002 -p 9003:9003 -p 9004:9004 -p 9005:9005 -p 9006:9006 -p 9007:9007 -p 9008:9008 -p 9009:9009 -p 9010:9010 -p 9011:9011 -p 9012:9012 -p 9013:9013 -p 9014:9014 -p 9015:9015 -p 9016:9016 -t -i storj-integration bash
```
You will then be given a 32-bit hex number, which is your container id.
Then you can then start and attach to the container with:
```bash
docker start -ai <container-id>
```
When it prompts with `root@<container-id>:~#`, the last step is to run the `start_everything.sh` script:
```bash
./scripts/start_everything.sh
```
As long as you want to connect to the bridge server, you need to keep this terminal window running.
You can then use pm2 commands to view logs etc.

#### 2. Register a Bridge User to Interact with Your Local Network
Now that you have integration’s local sandbox environment running, you need to create a user login and password to interact with the network and have the ability to view, create, list buckets, and upload and download files.

With node-libstorj installed, you can use the CLI to create this user with the command `storj -u http://localhost:6382 register`. This specifically registers a user with the local “integration” network that you’re running using localhost:6382. You will then be prompted for your bridge username (email) and password. The Storj CLI will then generate a secret key for you which it will use to encrypt your files, but first it will ask you what encryption strength to use (128, 160, 192, 224, or 256, with 256-bit being the strongest and recommended).

Once you input your secret key strength, you will be given your encryption key, a 24-word mnemonic, which you will later need for this project to decrypt any files that you upload to the Storj network-- so make sure to write it down and don’t leave it in a place where it can be hacked or stolen.

Now that you’ve registered a bridge user for your local network, you need to activate that user’s credentials. You can do this by going to your shell that’s running integration. Next to the `root@<container-id>:~#` prompt, you can enter the MongoDB shell by using the command `mongo`.

Then you can connect to the `storj-sandbox` database to look for users with the following commands:
```
db = connect(‘storj-sandbox’)
db.users.find({})
```
Your bridge user account that you created with the CLI should then be listed here. {{ picture here would be good }} You can see that the `activated` key is set to `false`. In order to set it to true, use the following command:
```
db.users.findOneAndUpdate({_id:’<your email>’}, {$set:{activated: true, activator: null}});
```

#### 3. Begin Creating an Express App
While not required, using the Express application generator may save some time in setting up your Express app because it creates an application skeleton and installs the Express command-line tool. To do this, use the command:
```
npm install express-generator -g
```
Within your project folder, you can then use the command `express --view=hbs storj-miniproxy`  to generate the express app and set its view engine to Handlebars.

Then install dependencies: `npm install --save dotenv express-handlebars multer`. We will talk about these dependencies later on as we continue to develop the app.

#### 4. Enabling Your App to Talk to Your Local Test Network
In order to let your app access your local test network (the one that you set up in Step 1), you need to set those variables in a way that your app can use then, while also protecting your password and encryption key (even though you’re only connecting your app to a local test network for now, you wouldn’t want to forget and later upload your login secrets to Github where anyone could use them and unlock your data!)

In order to solve this problem, we use the Dotenv module to protect our environment variables (which we npm installed in the last step).

Now, create a file named `.env` in the root of your project folder. In the `.env` file, you should have the following variables:
```
BRIDGE_URL=’http://localhost:6382’
BRIDGE_EMAIL=’<bridge email registered with the CLI>’’
BRIDGE_PASS=’<this is the password that you used when registering a user with the CLI>’
ENCRYPT_KEY=’<this is the 24-word mnemonic secret key that the CLI created for you>’
```
These are the only lines that your `.env` file will need.
In `app.js`, you’ll need to access these variables in order to set up your Storj environment used to access the bridge server. The Storj environment variables are like user login credentials to enable your access to the bridge.
```
// set up storj environment
const { Environment } = require('storj');
const storj = new Environment({
    bridgeUrl: process.env.BRIDGE_URL,
    bridgeUser: process.env.BRIDGE_USER,
    bridgePass: process.env.BRIDGE_PASS,
    encryptionKey: process.env.ENCRYPT_KEY,
    logLevel: 4
}
```
Also, I stored all of my route files in `routes/` instead of `app.js` for clearer organization.
