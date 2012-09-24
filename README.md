# rchook

In reponse to a Product->Archive in Xcode, automatically:

 1. Verify all git working directories are clean
 2. Bump build number to next even number
 3. Create .xcarchive file
 4. Do a `git commit` / `git tag` of main project and all sub-projects
 5. Bump build number to next odd number, do another `git commit`
 6. Copy .xcarchive and all source of all projects to external archive directory


## Why?

I submitted the wrong binary to Apple for [Theory Lessons](http://musictheory.net/buy/lessons) 1.2.  A week
later, it was approved.  This completely broke landscape orientation on iPad.  After pounding my forehead into
the table a few times (and resubmitting the correct binary), I decided to look at my submission process
and see if I could prevent this disaster from reoccuring.

Major issues with my former submission process:
 1. Xcode doesn't show the build number (CFBundleVersion) of archives, only the marketing version
 2. Even if Xcode showed the build number, my correct binary and my faulty binary had the same build number
 3. I didn't realize that I could re-sign binaries (change from App Store to Ad-hoc).
My testing process involved making an Ad-hoc build for [TestFlight](http://testflightapp.com) testing, and another binary for App Store distribution.  Talking with fellow developers, this appears to be a common misconception
 4. I previously used `agvtool` and `apple-generic` versioning to manage my build number.  At some point from Xcode 4.3 -> 4.5, `agvtool` stopped bumping my Info.plist files.


## Needed Features

After much coffee and pondering, I came up with the following feature set for my new build system:

 1. There should be a one-to-one relationship between Xcode archives and build numbers.  Each successfully
    built archive should have its own unique build number.
 2. The same archive/build number should be submitted to both TestFlight and the App Store.
 3. Development builds should never have the same build number as release.  For my purposes, it's ok if 
    different development builds share the same build number, as those builds only go on my devices.
 4. git tagging of release builds should be handled automatically
 5. A copy of all source code used to build the product, as well as the resulting xcarchive, should be stashed
    in a separate git repository (which is backed up to multiple servers).
 6. Xcode's Organizer's Archives window needs to show the build number, somehow.
 7. Everything should magically occur from Product->Archive in Xcode.  If anything goes wrong, Xcode stops the archive
    process and no .xcarchive is produced.


## Project Hierarchy

Both [Tenuto](http://musictheory.net/buy/tenuto) and [Theory Lessons](http://musictheory.net/buy/lessons) have one
main Xcode project, with multiple sub-project dependencies:

    Tenuto.xcodeproj (product=Tenuto.app)
        IXCore.xcodeproj (product=libIXCore.a)
        MTCore.xcodeproj (product=libMTCore.a)

    TheoryLessons.xcodeproj (product=TheoryLessons.app)
        IXCore.xcodeproj (product=libIXCore.a)
        MTCore.xcodeproj (product=libMTCore.a)
        SwiffCore.xcodeproj (product=libSwiffCore.a)

Each project is in its own git repository.  When I build Tenuto with CFBundleVersion=100, I want to tag all
three repositories with "Tenuto-100" (see Feature #4 from above)

Unfortunately, having a project hierarchy like this adds complexity to any build scripts.


## Xcode Scripts

Xcode has two ways of executing scripts: "Run Script" Build Phases and Scheme Actions.  My first attempt
used Build Phases exclusively.  Unfortuately, the earliest you can run a "Run Script" Build Phase is after
the internal CopyPlist phase, which is too late for modifying the Info.plist file (Feature #1).

My second attempt used Scheme Actions exclusively.  Unfortuately, there is no way to stop the build process
during a Scheme Action (`exit 1` doesn't work).

My third attempt used pre and post actions on the Archive Scheme of Tenuto, and Run Script Build Phases on 
each target.  Unfortuately, the Run Script Build Phases were running *before* the Archive pre-script.

It looks like the internal ordering of build phases and scheme actions for one of my projects (Tenuto) looks
something like this:

    Tenuto - Build Scheme pre-Action
        IXCore - Build Scheme pre-Action
            IXCore - Run Script Build Phase
        IXCore - Build Scheme post-Action

        MTCore - Build Scheme pre-Action
            MTCore - Run Script Build Phase
        MTCore - Build Scheme post-Action

        Tenuto - Run Script Build Phase
    Tenuto - Build Scheme post-Action

    Tenuto - Archive Scheme pre-Action
    Tenuto - Archive Scheme post-Action


Hence, the very first opportunity to run a script is the Build Scheme pre-action.
The last opportunity to run a script is the Archive Scheme post-action.
Each project can be cancelled during a Run Script Build Phase.

The configuration for all of this looks like:

Edit Scheme window for main project (Tenuto):

<img src="https://raw.github.com/musictheory/rchook/master/images/ss1.jpg" width="342" height="180"><br>
<img src="https://raw.github.com/musictheory/rchook/master/images/ss2.jpg" width="342" height="180"><br>

Build Phases window for **both main project and sub-projects**:

<img src="https://raw.github.com/musictheory/rchook/master/images/ss3.jpg" width="311" height="293"><br>


## The Actual Script (rchook)

Upon hitting Product->Archive in Xcode

 1. Tenuto Build scheme starts, `rchook` is called with `xcode-app-build-pre-action` argument
  1. Creates a fresh `/tmp/rchook` directory
  2. Modifies the main project's Info.plist to prepare for an archive release.
     Archives are given even build numbers.
  3. Creates a backup copy of the original Info.plist
  4. Creates an Info.plist with the next development build number (odd).  
  5. Creates `/tmp/rchook/cleanup.sh`, which is called to revert everything if we abort
 2. For each project (Tenuto, IXCore, MTCore), `rchook` is called with the `xcode-build-phase` argument
  1. Ensures that the git working directory is clean.  If not, exits out and causes Xcode
     to abort the archiving process.  This is needed since we will be commit to the git
     repository at the end of the archive process.
  2. Concatenates current `$PROJECT_DIR` into `/tmp/project_dirs`
 3. Tenuto Build scheme finishes
 4. Tenuto Archive scheme starts
 5. .xcarchive producted
 6. Tenuto Archive scheme finishes, `rchook` is called with the `xcode-app-build-post-action` argument
  1. Iterate over each project directory in `/tmp/project_dirs`.  Change the working directory and `git commit` and `git tag`
  2. Performs some trickery to set the comment of Xcode's archive to the build number
  3. Create a new directory on the file system, copies .xcarchive file, as well as source files for each project to the directory
  4. Copy the new Info.plist created in Step 1.4 to the main project's directory
  5. Commit the new Info.plist using `git commit`


## Resources

These resources were helpful when developing rchook:
http://stackoverflow.com/questions/9855955/xcode-increment-build-number-only-during-archive
