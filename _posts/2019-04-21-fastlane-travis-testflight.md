---
layout: post
title: Fastlane + Travis + TestFlight Tutorial
filename: "2019-04-21-fastlane-travis-testflight.md"
---

This blog post is different from other ones. **I write it mostly for my future self.** I worked on setting up the CI pipeline for a new project recently. If you've ever done it, you know it's usually a painful experience.

<center>
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">I now understand why <a href="https://twitter.com/travisci?ref_src=twsrc%5Etfw">@travisci</a> gives the first 100 builds for free. It&#39;s roughly the number of tries you need before you make it properly deploy builds to TestFlight 🤦‍♂️</p>&mdash; Vojta Stavik (@vojtastavik) <a href="https://twitter.com/vojtastavik/status/1118988570197409793?ref_src=twsrc%5Etfw">April 18, 2019</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</center>

The biggest problem is that this kind of work is not something an average developer would do every week; not even every month. **I write this blog post to capture my experience so I have a step-by-step tutorial to use in the future.**

<!-- more -->

---

### 1. Create a new Apple developer account for CI

Start with creating a new Apple developer account you will use for CI. I like using format `ci@VojtaStavik.com`.

Technically, you can use your personal existing account, too. However, I like having a separate account just for the CI:
- The account **mustn't** have the 2-factor authentication enabled.
- When you work in a team, everyone should have access to the account. **You probably don't want to share you AppleID password with your coworkers.**
- You can give the account just the **minimal set of permissions needed.** For example, it should be allowed to upload new builds, but it doesn't need permissions to create or delete apps in AppStore connect.

Don't forget to add the new account to your developer organization.

---

### 2. Set up Fastlane

**2.1** Follow the official [Fastlane setup guide](https://docs.fastlane.tools/getting-started/ios/setup/) for the initial installation.

**2.2** Open `fastlane/Appfile` and add the following details. _Of course, with the correct values for your project_.
```ruby
app_identifier("app.VojtaStavik.com") # The bundle identifier of your app
apple_id("ci@VojtaStavik.com") # The email address of your "CI" account from step 1

itc_team_id("123456789") # App Store Connect Team ID
team_id("ABCD1234") # Developer Portal Team ID
```

**2.3** We will use Fastlane's [match](https://docs.fastlane.tools/actions/match/#match) to synchronize developer profiles and certificates across multiple devices, including CI. **Create a new GitHub repository** where we're going to store them.

You can name it: `[Project name]-certificates`.

**2.4** Follow the setup guide of [match](https://docs.fastlane.tools/actions/match/#match) and use the GitHub repo created in the previous step.

You have to provide a password to encrypt the repository. Write it down; we will need it later.

**2.5** Open `/fastlane/Fastfile` and remove the auto-generated content. We start with **defining a lane for running tests**:
```ruby
default_platform(:ios)

xcode_project_path = "________.xcodeproj" # Path to the xcodeproj file

# Always update fastlane to the latest version
update_fastlane

desc "Run tests"
lane :test do
  run_tests(
    project: xcode_project_path,
    scheme: "___________", # The name of the scheme you want to test. Usually, it's similar to your project name.
    devices: ["iPhone Xs (12.2)"] # The device(s) you want to run the tests on.
  )
end
```

_Note:_ If you're not sure which devices to use, you can run `xcrun simctl list`. This command prints the list of all available devices on the given machine.

**2.6** Then we add a **lane for deploying a new build to TestFlight**:
```ruby
desc "Deploy a new build to TestFlight"
lane :deploy do
  # Set the build number to the current date (UTC, 24h) -> "20190419.0510"
  build_number = Time.new.utc.strftime("%Y%m%d.%H%M")
  increment_build_number(
    xcodeproj: xcode_project_path,
    build_number: build_number
  )

  # Instal the required certificates and profiles
  setup_travis
  match(type: "appstore", readonly: true)

  # Build and deploy to TestFlight
  build_app(export_method: "app-store")
  upload_to_testflight
end
```

_In theory_, this should be all we need to make it work 😅. Unfortunately, at the time this post is written, **I couldn't make it work for Xcode 10.2 with automatic codesigning enabled.**

One solution would be to switch to manual codesigning. I didn't want to do that.

Another possible workaround is to temporarily disable automatic codesigning and set the signing details manually.

I put the following code between the `match` and `build_app` steps:

```ruby
  # Temporarily disable automatic code signing
  disable_automatic_code_signing(path: xcode_project_path)

  update_project_team(
    path: xcode_project_path,
    teamid: "ABCD1234" # Use  your developer team ID
  )

  # The "match" step sets environment variables with paths for all provisioning profiles.
  # You can find the exact name of the variable in the log generated by "match".
  provisioning_profile_path = ENV["sigh____________________profile-path"]

  # The code signing identity you want to use
  signing_identity = "iPhone Distribution: _________ (_________)"

  update_project_provisioning(
    xcodeproj:  xcode_project_path,
    profile: provisioning_profile_path,
    build_configuration: "Release",
    code_signing_identity: signing_identity
  )
```

As the very last step of the lane, we re-enable automatic codesigning. You don't need to do it if you deploy new builds _only_ from CI.
```ruby
  # Enable back automatic code signing
  enable_automatic_code_signing(path: xcode_project_path)
```

You can test both lanes locally by running:
```shell
$ bundle exec fastlane test
$ bundle exec fastlane deploy
```

---

### 3. Set up Travis

We want Travis to run tests for every pull request. Additionally, when there's a change to `master`, we want to run the tests and deploy a new TestFlight build.

**3.1** Open [TravisCI website](https://travis-ci.com) and grant Travis access to your project repository. **Don't forget to add the repository with profiles and certificates, too.**

**3.2** Open the Travis settings page for your project repo.

Change the General settings to build PRs and pushed branches:
![Travis general settings](/images/fastlane-travis-testflight/travis-general-settings.png)

Scroll down and add two new environment variables:

`FASTLANE_PASSWORD` - the password to your CI Apple developer account **(step 1)**.
`MATCH_PASSWORD` - the password used to encrypt the certificates repo **(step 2.4)**.

![Travis environment variables](/images/fastlane-travis-testflight/travis-env-variables.png)

**3.3** Create a new file in the root of your repository and name it `.travis.yml`.

Open it and add the following:
```yml
language: objective-c # even when you use Swift, this should still be objective-c
osx_image: xcode10.2 # or another Xcode version you use

branches:
  only:
  - master

before_install:
  - bundle update

script:
  # Exit immediately if one of the commands fails
  - set -e

  # Run tests
  - bundle exec fastlane test

  # If not pull request, deploy a new TestFlight build
  - if [ "$TRAVIS_PULL_REQUEST" == false ]; then
      bundle exec fastlane deploy;
    fi;
```

---

That's it 🎉 I hope this tutorial will be helpful and save you a couple of hours of headaches during the initial setup.
