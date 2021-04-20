# Installation

Before you begin, it is recommended to understand exactly how this project works. Knowing what is happening at each point will help you troubleshoot any issues far better. Check out the [How does this all work?](DETAILS.md) page.

There are multiple ways to install this web service:

- On your phone
- On your computer, without requiring HTTPS certificate or port forwarding
- On your server, with HTTPS and open port 443

But in all cases, you first need a builder.

For a video tutorial, [click here](https://www.youtube.com/watch?v=jZhECb8baWA). Note that you still need this written guide - use the video as an addition, but not substitution for the instructions below.

## 1. Builder

You can create a builder in one of two ways:

- **Use a Continuous Integration (CI) service** such as GitHub Actions or Semaphore CI. This method is the easiest, fastest, and most recommended way to make a builder. Head over to [ios-signer-ci](https://github.com/SignTools/ios-signer-ci) and follow the instructions.

- **Use your own Mac machine**. This method is only recommended if you already have a server Mac, you are somewhat experienced in server management, and you would like to host your truly own builder. Go to [ios-signer-builder](https://github.com/SignTools/ios-signer-builder) for instructions.

Once you have made your builder, proceed below.

## 2. Web service configuration

It's easier if you use your personal computer for the initial configuration. This guide assumes you are doing that.

### 2.1. Configuration file

You need to create a configuration file which links the web service to your builder.

1. Download the correct [binary release](https://github.com/SignTools/ios-signer-service/releases) for your computer
2. Run it once - it will exit immediately, saying that it has generated a configuration file
3. In the same folder as the binary, you will find a new file `signer-cfg.yml` - open it with your favorite text editor and configure the settings using the explanations below. The lines that start with a hashtag `#` are comments, you do not need to touch them.

> :warning: **Don't forget to set "`enable: true`" on the builder that you are configuring!**

```yml
# here you define the builder you created in the previous section
# configure only the one that matches yours
builder:
  # GitHub Actions
  github:
    enable: false
    # the name you gave your builder repository
    repo_name: ios-signer-ci
    # your GitHub profile/organization name
    org_name: YOUR_ORG_NAME
    workflow_file_name: sign.yml
    # your GitHub personal access token that you created with the builder
    token: YOUR_GITHUB_TOKEN
    ref: master
  # Semaphore CI
  semaphore:
    enable: false
    # the name you gave to your Semaphore CI project
    project_name: YOUR_PROJECT_NAME
    # your Semaphore CI profile/organization name
    org_name: YOUR_ORG_NAME
    # your Semaphore CI token that you got when creating the builder
    token: YOUR_SEMAPHORE_TOKEN
    ref: refs/heads/master
    secret_name: ios-signer
  # your own self-hosted Mac builder
  selfhosted:
    enable: false
    # the url of your builder
    url: http://192.168.1.133:8090
    # the auth key you used when you started the builder
    key: SOME_SECRET_KEY
# the public address of your server, used to build URLs for the website and builder
# must be valid HTTPS or web install (OTA) won't work!
# leave untouched if you don't know what this means - use a tunnel provider instead
server_url: https://mywebsite.com
# where to save data like apps and signing profiles
save_dir: data
# apps older than this time will be deleted when a cleanup job is run
cleanup_mins: 10080
# how often does the cleanup job run
cleanup_interval_mins: 30
# apps older than this time will be marked as failed
# this should also match the job timeout in the builder
sign_timeout_mins: 10
# this protects the web ui with a username and password
# definitely enable it if you left "server_url" empty and are using a tunnel provider
basic_auth:
  enable: false
  username: "admin"
  # don't forget to change the password
  password: "admin"
```

### 2.2. Signing profiles

There are two types of signing profiles:

- **Certificate + provisioning profile**

  If you have a paid developer account, it is highly recommended to use this method. Doing so will save you from a lot of limitations. To get a provisioning profile (`.mobileprovision` file), [create one](https://developer.apple.com/library/archive/recipes/ProvisioningPortal_Recipes/CreatingaDevelopmentProvisioningProfile/CreatingaDevelopmentProvisioningProfile.html) from your developer portal and download it. You will probably want it to be a `Development` type and not `Distribution`, so that you can have a `wildcard` application identifier and app debugging entitlement (`get-task-allow`). For the differences, check the [FAQ](FAQ.md) page. Also don't forget to [register the UDID](https://developer.apple.com/library/archive/recipes/ProvisioningPortal_Recipes/AddingaDeviceIDtoYourDevelopmentTeam/AddingaDeviceIDtoYourDevelopmentTeam.html#//apple_ref/doc/uid/TP40011211-CH1-SW1) of each device that you want to sideload to. Read ahead on how to get your certificate.

- **Certificate + developer account**

  If you don't have a paid developer account, this is your only option. Make sure to read and understand the limitations in the [FAQ](FAQ.md) page before you proceed. Read ahead on how to get your certificate.

The certificate is a file with an extension `.p12`. To obtain it, follow the instructions below:

**On macOS:** Install [Xcode](https://developer.apple.com/xcode/) and open the `Account Preferences` (A). Sign into your account using the plus button. Select your account and click on `Manage Certificates...`. In the new window (B), click the plus button and then `Apple Development`. Click `Done`. Now open the `Keychain` app (C). There you will find your certificate and private key. Select them by holding `Command`, then right-click and select `Export 2 items...`. This will export you the `.p12` file you need.

<table>
<tr>
    <th>A</th>
    <th>B</th>
    <th>C</th>
</tr>
<tr>
    <td><img src="img/6.png"/></td>
    <td><img src="img/7.png"/></td>
    <td><img src="img/5.png"/></td>
</tr>
</table>

**On all other platforms:** There is no official way to do this. You should be able to use a third-party signing tool like [AltStore](https://altstore.io/) and then you should be able to find the certificate in its app data (`Program Data` on Windows). However, this has not been tested.

Once you have your certificate and optionally provisioning profile, you need to create the correct folders for the service to read them:

1. Create a new folder named `data` (if you changed `save_dir` in the config above, use that)
2. Create another folder named `profiles` inside of it
3. Create a new folder named `my_profile` inside of `profiles`. You can use any profile name here, this will be the ID of your signing profile
4. Put the signing related files inside here. Read ahead to see what they should be named
5. Repeat the steps above for each signing profile that you want to add

> :warning: **You need to match the files names exactly as they are shown below. For an example, your certificate must be named exactly `cert.p12`. Be aware that Windows may hide the extensions by default.**

- **Certificate + provisioning profile**

  ```python
  data
  |____profiles
  | |____my_profile                # any unique string that you want
  | | |____cert.p12                # the signing certificate
  | | |____cert_pass.txt           # the signing certificate's password
  | | |____name.txt                # a name to show in the web interface
  | | |____prov.mobileprovision    # the signing provisioning profile
  | |____my_other_profile
  | | |____...
  ```

- **Certificate + developer account**

  ```python
  data
  |____profiles
  | |____my_profile                # Or what you named your profile
  | | |____cert.p12                # the signing certificate
  | | |____cert_pass.txt           # the signing certificate's password
  | | |____name.txt                # a name to show in the web interface
  | | |____account_name.txt        # the developer account's name (email)
  | | |____account_pass.txt        # the developer account's password
  | |____my_other_profile
  | | |____...
  ```

That's all the initial configuration! To recap, you now have the following configuration files:

- `data` folder (or whatever you named it in `save_dir` in the config)
- `signer-cfg.yml` file

## 3. Web service installation

You can install the web service on your computer, on a server, or on your phone. The device that you choose will have to be connected to the internet in order for anybody to use the service.

### 3.1. Self-hosting on phone

#### 3.1.1. Preparing

1. Get the [iSH](https://ish.app/) app on your phone. Open the app, and when text appears, close it.
2. Move the configuration files you made in sections `2.1.` and `2.2.` of this guide to your phone. You can use any method, like [iTunes](https://www.apple.com/us/itunes/) or [iMazing](https://imazing.com/). It doesn't matter where you put the files as long as you can access them from the Files app on your phone.
3. Open the Files app on your phone.
4. Swipe to the right until you go to the `Browse` screen.
5. In the top-right corner, click on the three dots and select `Edit`.
6. Enable (toggle) the `iSH` entry under `Locations`.
7. Move the files you just copied from your computer to the `iSH` location you just enabled, inside the folder `root`.
8. Open the `iSH` app again.
9. Type `ls` and press enter. If you did everything correctly, you should see the names of the files you just moved in.
10. Type `apk add curl` and press enter.

#### 3.1.2. Installing

1. Type the following command and press enter:
   ```bash
   curl -sL git.io/ios-signer-ish | sh
   ```

#### 3.1.3. Running

1. Type the following command and press enter:

   ```bash
   ./start-signer.sh
   ```

   > :warning: **When iOS asks you to grant location permission to iSH, click "Always Allow". The location data is not used for anything, but the permission allows the service to keep running in the background if you minimize iSH or lock your phone.**

2. When the service finishes loading, look for a line similar to this:

   ```log
   11:51PM INF  state="obtained public url" url=https://aids-woman-zum-summer.trycloudflare.com
   ```

   `https://xxxxxxxxxxxx.trycloudflare.com` is the public URL of your service. That's what you want to open in your browser. Congratulations!

   Due to Apple's strict background process policy, iSH will get killed if it uses more than "80% cpu over 60 seconds". This can break any part of the service. To make sure it doesn't happen:

   > :warning: **Use Safari browser. Whenever you are waiting for something to complete, such as app upload, signing, or installation, open iSH in the foreground. Safari will continue in the background.**

   Watch the logs to know when the operation is complete. You can periodically switch between Safari and iSH to check as well, just don't leave iSH in the background for more than 30 seconds.

#### 3.1.4. Updating

Just repeat the `Installing` section.

### 3.2. Self-hosting on computer or server

#### 3.2.1. Installing

You have two options:

**Normal**

1. Download the correct [binary release](https://github.com/SignTools/ios-signer-service/releases) (if this is a different computer than you used before)
2. Move the configuration files you made in sections `2.1.` and `2.2.` of this guide to the same folder as the binary you just downloaded

**Docker**

1. Use the official [Docker image](https://hub.docker.com/r/signtools/ios-signer-service)
2. Move and [mount](https://docs.docker.com/storage/volumes/) the configuration files from sections `2.1.` and `2.2.`:
   - `./signer-cfg.yml:/signer-cfg.yml`
   - `./data:/data` (or whatever you set in `save_dir`)

#### 3.2.2. Running

For reference, these are the default arguments that will be used:

- The listening port is 8080. You can change this with the argument `-port 1234`
- The listening host is all (0.0.0.0). You can change this with the argument `-host 1.2.3.4`
- For more, use `-help`

The web service cannot work by itself. You have two options:

**Reverse proxy** - secure, fast, reliable, but harder to set up

- Requires publicly accessible port 443 (HTTPS)
- Requires domain with valid certificate
- Requires manual configuration of reverse proxy with your own authentication.
- Don't protect the following endpoints:
  ```
  /apps/:id/
  /jobs
  /jobs/:id
  ```
  where `:id` is a wildcard parameter.

**Tunnel provider** - less secure, slower, but quick and easy to set up

[ngrok](https://ngrok.com/) and [Cloudflare Argo](https://blog.cloudflare.com/a-free-argo-tunnel-for-your-next-project/#how-can-i-use-the-free-version) are supported as tunnel providers. The latter will be demonstrated in this guide. Run the signer service with `-help` to see alternative details.

1. Download the correct [cloudflared](https://github.com/cloudflare/cloudflared/releases/latest) binary for your computer.
2. **Every time before** starting your service, execute the following command and keep it running:
   ```bash
   cloudflared tunnel -url http://localhost:8080 -metrics localhost:51881
   ```
3. Then start your service with the following command:
   ```bash
   ios-signer-service -cloudflared-port 51881
   ```
4. When the service finishes loading, look for a line similar to this:
   ```log
   11:51PM INF  state="obtained public url" url=https://aids-woman-zum-summer.trycloudflare.com
   ```
   `https://xxxxxxxxxxxx.trycloudflare.com` is the public URL of your service. That's what you want to open in your browser. Congratulations!
