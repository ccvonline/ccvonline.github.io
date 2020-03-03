---
layout: post
title:  "Xamarin Bindings"
date:   2020-03-02 10:29:05 -0700
categories: xamarin bindings c# ios android
visible: 1
---
# The Project
I work as a software engineer at a church where we have a growing population of native Spanish speakers. Recently, we decided it would be nice to offer a live Spanish translation of our service for those that either didn't speak English or simply preferred to listen in Spanish.

Since we have a Mobile App for both Android and iOS, we wanted to take advantage of that and allow people who wanted to listen to access it through our app. We would have a translator speaking into a device that broadcasts on our public Wifi network, and users would launch our Mobile App, connect to our Wifi and listen to the stream in real time.

After R&Ding several solutions, we settled on a device from a company named [AudioFetch](https://www.audiofetch.com/). The hardware was reliable and they offered an SDK in the form of native iOS and Android libraries you link in and use. Perfect!

So this project began with two files and a straight forward task. Take `libAudioFetchSDK.a` and `afaudiolib.aar` and come out with `AudioFetchiOS.dll` and `AudioFetchDroid.dll`. On paper, this simply requires using Visual Studio to create C# bindings for both iOS and Android respectively. They already have the project templates for this, so what could be simpler?

I flipped a coin and it landed on Android, so down the rabbit hole I went...

## Android Bindings

# A simple Java Android App
I figured the best way to start would be to simply get their library working in a native Android app written in Java. Generating bindings would be enough trouble on its own, and I didn't need the extra complexity of not knowing how their SDK worked.

In the AudioFetch documentation, they tell you to create a new Android project, and include the following dependencies in your gradle build file:

    dependencies {
        implementation fileTree(dir: 'libs', include: ['*.jar'])

        implementation 'com.android.support:appcompat-v7:28.0.0'
        implementation 'com.android.support.constraint:constraint-layout:1.1.3'

        // Audiofetch SDK Dependancies
        implementation 'io.reactivex.rxjava2:rxjava:2.2.0'
        implementation 'io.reactivex.rxjava2:rxandroid:2.0.1'
        implementation 'com.jakewharton.rxrelay2:rxrelay:2.0.0'
        implementation 'com.google.guava:guava:19.0'

        // Audiofetch SDK library
        implementation(name: 'afaudiolib', ext: 'aar')
    }

and basic usage looks like this:

    AFAudioService.api().outMsgs()
    .asFlowable()
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe( msg -> {
    if (msg instanceof AfApi.ChannelsReceivedMsg) {
            AfApi.ChannelsReceivedMsg  crMsg = (AfApi.ChannelsReceivedMsg) msg;
            onChannelsReceivedEvent(crMsg);
        }
    });

Looking at the dependencies, I noticed they're using `RXJava` as a messaging system. This meant their library had additional dependencies that I was going to have to figure out how to bind in. But that was a problem for future me.

In a pretty short amount of time I had a prototype app working, in Java, using their library.
It was time to generate C# bindings!

# Generating C# Bindings
The first step was to create a new Android Bindings project in Visual Studio.

I added `afaudiolib.aar` and set its build action to `LibraryProjectZip`, since this was the main file I wanted bindings generated for.

# Missing dependency Jars
Next, I needed to add the Jar files that `afaudiolib.aar` depends on, which were listed in the gradle config above. 
But wait...the only place I knew to look for Jars was the `libs` folder in my project.

In that `libs` folder, the only file I had was `afaudiolib.aar`. So where are the other Jars I needed?
![alt text](/images/xaramin-bindings-3-2-2020/libs-1.png "Libs Folder")

Notice the definition of the dependencies for the missing Jars:

    implementation 'io.reactivex.rxjava2:rxjava:2.2.0'
    
As I learned, that instructs gradle to go download the Jar AT BUILD TIME and link it in. Which means there's no physical file for me to go get. Hrm. I need those Jars.

After scouring the internet, I came across this extremely helpful gradle action:

<https://gist.github.com/n-belokopytov/d44949590748b096c1a497008b761d04>

This is a gradle dependency copier, which will take the downloaded Jars, resolve their names, and store them in a folder on the file system!

After figuring out how to use gradle at a command line, I typed `gradlew copyDependenciesDebug` and got the following output:

    // Copying dependency:
    // /Users/jered/.gradle/caches/modules-2/files-2.1/io.reactivex.rxjava2/rxjava/2.2.0/2c622e8f1cab8d5871d32bb2023af515216056d8/rxjava-2.2.0.jar

![alt text](/images/xaramin-bindings-3-2-2020/gradle-output.png "Gradle Output")

Yay! This meant the dependency Jars had been downloaded and copied out to a location on the file system. I went to each folder and grabbed my four dependencies.

Now, before jumping right back into the bindings project, I took a detour and reconfigured my prototype Java app to use these copied Jars instead of the download-at-build-time versions. I didn't want to just assume things would work.

I changed the dependency includes to use local Jars instead of downloading them, like so:

This "download at build time"

    implementation 'io.reactivex.rxjava2:rxjava:2.2.0'

became "look for a local version" 

	implementation(name: 'rxrelay-2.0.0', ext: 'jar')

The dependencies now looked like this

	dependencies {
    	implementation 'com.android.support:appcompat-v7:28.0.0'
    	implementation 'com.android.support.constraint:constraint-layout:1.1.3'

    	// Audiofetch SDK Dependancies
    	implementation(name: 'reactive-streams-1.0.2', ext: 'jar')
    	implementation(name: 'rxjava-2.2.0', ext: 'jar')
    	implementation(name: 'rxandroid-2.0.1-api', ext: 'jar')
    	implementation(name: 'rxrelay-2.0.0', ext: 'jar')
    	implementation(name: 'guava-19.0', ext: 'jar')

    	// Audiofetch SDK library
    	implementation(name: 'afaudiolib', ext: 'aar')
	}

and my libs folder now looked like this
![alt text](/images/xaramin-bindings-3-2-2020/libs.png "Libs Folder")

I then built and ran the app, and it worked as before, but now with all Jars stored locally!
NOW I had everything I needed to resume generating the bindings.

# Binding Compiler Errors
I added the additional Jars to the Jars folder, and because they needed to actually be bundled into the DLL, set their build action to `EmbeddedReferenceJar`
![alt text](/images/xaramin-bindings-3-2-2020/ref-jar.png "Embedded Reference Jar")

I hit rebuild and waited... 5 errors.

The Android Binder works in a two step process. The first step parses the libraries flagged for parsing (in this case just `afaudiolib.aar`) and creates matching C# wrapper classes for them. The second step compiles the generated C# code and links it into a DLL. If you get compiler errors, it's most likely because the first step failed to correctly generate the C# wrapper.

The fix for these issues is to use the `Metadata.xml` file and implement transforms that tell the binder how to generate the C# code for trouble areas. This file uses the [XPath](https://www.w3.org/TR/xpath/) syntax, which means if you're like me, it's yet another thing you'll have to go get familiar with just to fix these errors.

The first error was an illegal naming convention in C#, where a class member cannot be the same name as its owning class.

    Error CS0542: 'Channel': member names cannot be the same as their enclosing type (CS0542)

To fix this issue, I had to tell the binder to use a different name for the C# wrapper.

    <attr path="/api/package[@name='com.audiofetch.afaudiolib.dal']/class[@name='Channel']/field[@name='channel']" name="name">ChannelIndex</attr>

Unfortunately when I ran the DLL with this fix, I got a runtime exception whenever I referenced `ChannelIndex`. The exception said `ChannelIndex` was a non-existent property in Java. 

What I failed to realize was `name="name"` told the binder to generate a C# wrapper called `ChannelIndex` bound to a Java member ALSO called `ChannelIndex`, like so:

	[Register ("ChannelIndex")]
	public int ChannelIndex {
		get {
			const string __id = "ChannelIndex.I";

			var __v = _members.InstanceFields.GetInt32Value (__id, this);
			return __v;
		}
	}

The problem is `ChannelIndex` doesn't exist in Java. In Java it's just called `channel`.

It turned out I needed the C# WRAPPER to be `ChannelIndex`, but have it lookup the underlying Java property by `channel`.
By using `name="managedName"` we told the binder exactly that. I changed the transform to be

	<attr path="/api/package[@name='com.audiofetch.afaudiolib.dal']/class[@name='Channel']/field[@name='channel']" name="name">ChannelIndex</attr>

and got

	[Register ("channel")]
	public int ChannelIndex {
		get {
			const string __id = "channel.I";

			var __v = _members.InstanceFields.GetInt32Value (__id, this);
			return __v;
		}
	}

and it worked!


The next set of errors were all missing interface implementations, along the lines of:

    Error CS0535: 'Apb' does not implement interface member 'IComparable.CompareTo(Object)' (CS0535)


These were a little more straight forward. I simply needed to instruct the binder to generate bindings for the missing types.

    <add-node path="/api/package[@name='com.audiofetch.afaudiolib.dal']/class[@name='Apb']">
        <method name="compareTo" return="int" abstract="false" native="false" synchronized="false" static="false" final="false" deprecated="not deprecated" visibility="public">
            <parameter name="p0" type="java.lang.Object" />
        </method>
        
        <method name="compare" return="int" abstract="false" native="false" synchronized="false" static="false" final="false" deprecated="not deprecated" visibility="public">
            <parameter name="p0" type="java.lang.Object" />
            <parameter name="p1" type="java.lang.Object" />
        </method>
    </add-node>

With that in place, the binding compiled! My next step was to put this new `AudioFetchDroid.dll` into a simple C# app and see if it worked.

# Java compiler errors
After setting up a basic Xamarin Android C# app and adding the DLL as a reference, I compiled. 

And error.

    Error JAVAC0000:  error: OnReportCompletedListener is not public in ReportController; cannot be accessed from outside package com.audiofetch.afaudiolib.bll.colleagues.ReportController.OnReportCompletedListener

After doing some research, I understood the problem to be an access modifier issue with the way `IReportCompletedListener` was being generated. I spent a little time attempting to transform its access to be public to no avail, and finally realized it wasn't necessary to transform, because I had no need of referencing it in my DLL.

So my "fix" was to update `Metadata.xml` and tell the binder not to generate bindings for this class.

    <remove-node path="/api/package[@name='com.audiofetch.afaudiolib.bll.colleagues']/class[@name='ReportController']" />
As a side note, I'd love feedback on how to correctly fix this in the future.

With these errors gone, the simple C# app compiled and was ready to run.

# Building the simple C# app
It was time to begin implementing functionality that used the DLL.
It didn't take long for the next issue to become apparent.

# Dependency libraries surfaced in C#
Note earlier that I said usage of AudioFetch in Java looked like this:

    AFAudioService.api()
                .outMsgs()
                .asFlowable()
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(
                    msg -> {
                        if (msg instanceof AfApi.ChannelsReceivedMsg) {
                            AfApi.ChannelsReceivedMsg  crMsg = (AfApi.ChannelsReceivedMsg) msg;
                            onChannelsReceivedEvent(crMsg);
                        }
                    });

As I began to type the C# equivalent to use the AudioFetch DLL, I ran into an issue.

    AFAudioService.Api(). //Yay!
                OutMsgs(). //Ok!
                AsFlowable(). //This didn't exist...

The problem was `AsFlowable()` didn't exist in C#. Why not? Because the method wasn't there.

What existed in C# was the highest level API for AudioFetch. And while yes, I did link in the Jars that AudioFetch needed, its method `OutMsgs()` returns a straight RxJava object, which means it needed exposure to C# as well.

Before I even considered building bindings for these myself, I checked nuget, and thankfully ran across packages that ALREADY BOUND THESE. 

[Thank you NAXAM!](https://github.com/naxam)

Now, I could have just added these packages to my C# app, but the problem was `AsFlowable()` is a method on the object returned by `OutMsgs()`. And `AsFlowable()` was never bound because at BINDING TIME the binder wasn't aware of the java type (because the Jar was being included as a bundled resource and not a reference for binding.)

So if I had just included the packages in the simple C# app, I basically would have been able to do:

    AFAudioService.Api(). //Yes
                OutMsgs(). //Yes
                AsFlowable(). //No, because it was never bound.
                .ObserveOn //Yes, because it was part of RxJava

Which obviously won't work. Back to the bindings project.

Using nuget I added these IN THE BINDINGS PROJECT. Since these bindings obviously include the Jar files, I also removed those specific Jar files from my Jar references. (Otherwise we'll get duplicate symbol link errors.)

This left me with three Jar files (`afaudiolib.aar`, `guava-19.0.jar`, and `rxrelay-2.0.0.jar`) and two C# references (`Naxam.RxAndroid.Droid` and `Naxam.RxJava2.Droid`)
![alt text](/images/xaramin-bindings-3-2-2020/droid-binding-libs.png "Android Bindings")

Successful compile! And as I hoped, the binder saw the C# binding references and was able to generate the missing C# bindings for the lower level AudioFetch API (`AsFlowable()`) in addition to providing me with the DLLs for RXJava and RXAndroid.

# Bringing it all together
Now back in the simple C# app, I was able to add references to `RXAndroid` and `RXJava`, as well as `AudioFetchDroid`.

After cleaning the solution, I began typing, and was able to type this:

    AFAudioService.Api().
        OutMsgs().
        AsFlowable().
        ObserveOn(IO.Reactivex.Android.Schedulers.AndroidSchedulers.MainThread())
        .Subscribe(this);

It compiled, it linked, and it ran! Finally I had successfully created C# bindings for AudioFetch and its dependency libraries, and was able to use it in a test app.

A this point, with it working entirely in C#, I felt confident enough to move these DLLs into our production app, and we are now sucessfully streaming audio from AudioFetch!

Next up, iOS.

## iOS Bindings

# Building a simple iOS app
I began with the same process that proved useful for Android, and went about building a simple Objective-C app that used the `libAudioFetchSDK.a` library.

Similar to Android, they have some basic setup documentation, and it says:
1. Setup a new Xcode Project
2. Add the AudioFetch SDK headers
3. Add the following libraries to the "Link Binary with Libraries" Build Phase: `libAudioFetchSDK.a`, `MediaPlayer.framework`, `libresolv.tbd`
4. Add `lstdc++` to the linker flags in Build Settings. (Basically include the C++ standard library)
5. Use CocoaPods to include the dependency library `AFNetworking`.

Step five was confusing for me on multiple levels. First, I'd never used CocoaPods before. I assumed it was a dependency manager for Xcode? (It is.) and I assumed that even though `AFNetworking` started with AF, like "AudioFetch" abbreviates to, this was just an unfortunate and extremely confusing coincidence. (Yup.)

After getting a crash course in setting up and using CocoaPods, I had my app building. 

Usage looked pretty straight forward, with calls like `[[AudioManager sharedInstance] startAudio];` 

As I put together basic playback code, I was mindful of watching out for any API that surfaced `AFNetworking`, and thankfully didn't run into any. So unlike Android it appeared that `AFNetworking` was only used internally by their library, and I wouldn't have to worry about generating bindings for this. As a bonus, if I do want to utilize any `AFNetworking` in my app, I noticed that once again [NAXAM](https://github.com/naxam) comes to the rescue with [an `AFNetworking` C# binding.](https://github.com/NAXAM/afnetworking-ios-binding)

So! The sample app is building. It's time to generate some C# bindings.

# Generating C# iOS bindings
A quick Google search landed me at the [Microsoft Objective-C Binding Documentation](https://docs.microsoft.com/en-us/xamarin/ios/platform/binding-objective-c/).
It's never an encouraging sign when the front page has a one hour YouTube tutorial. Especially one filmed in 2015. Since I had success going step by step with Android, I wanted to do the same here, so I watched the video. It ended up being worthwhile because it helped show me what unknowns I would need to account for.

Like Android, there is a project for generating iOS bindings. Unlike Android, there were no XML files. Instead I had two C# files: `APIDefinitions.cs` and `StructsAndEnums.cs`. According the documentation, I would use a tool called `Objective Sharpie` to parse the AudioFetch headers, and that would output an `APIDefinitions.cs` and `StructsAndEnums.cs` that contained most of the bindings for me. Basically it autogenerates best-guess C# bindings, and then leaves it to me to fix any compiler errors.

# Trying the "Walkthrough: Binding an iOS Objective-C Library"
If you want your developers to use a walkthrough you provide, starting out your documentation with a warning like this is a good way to do it.
![alt text](/images/xaramin-bindings-3-2-2020/warning.png "Objective Sharpie Warning")
I don't consider myself to have 'advanced knowledge' of much, so off to their [Binding an iOS Objective-C Library](https://docs.microsoft.com/en-us/xamarin/ios/platform/binding-objective-c/walkthrough?tabs=macos) walkthrough I went.

This proved useful because it taught me how to generate a static library in Xcode, which was something I would end up needing later for `AFNetworking`. Unfortunately when attempting to build the static library, it became apparent that this walkthrough was about 4 years out of date and I wasn't going to be able to proceed. (Literally, because it's not compatible with current SDKs.)

So I decided to throw caution to the wind and just try and run `Objective Sharpie` against the AudioFetch headers.

Usage for `Objective Sharpie` said to go to the folder containing my headers and run a simple command:
	
	sharpie bind -output=AudioFetch --namespace=AudioFetch --sdk=iphoneos13.2 include/audioFetchSdk/AudioManager.h

Running this resulted in: `zsh: command not found: sharpie` What? Back to the documentation.
Oh. This tool isn't included as part of the Visual Studio for Mac / Xamarin installation. It's a separate download linked within the documentation for `Objective Sharpie`.

After downloading it, I ran `sharpie update` to verify I was up-to-date (per their documentation) and it said I was! Now we're ready!

I again ran the sharpie command above, and started to see output! Unfortunately that output included an error stating it couldn't generate `APIDefinitions.cs`, due to something related to `macOS Catalyst`.

I recognized Catalyst as the name of [Apple's project for bringing iPad apps to macOS](https://developer.apple.com/mac-catalyst/), and immediately wondered if this supposedly up-to-date version of `Objective Sharpie` was actually the most up-to-date version. Sure enough, after doing some additional searching, I discovered I had downloaded version 3.4, and there existed a 3.5.22. It turns out they changed the update channel that newer versions update with, so I had to find an alternative download link for a version newer than 3.4, and then update THAT version to get to the latest 3.5.22.

Now with the latest version installed, I ran the sharpie command again, and saw a successful `APIDefinitions.cs` and `StructsAndEnums.cs` generated!
![alt text](/images/xaramin-bindings-3-2-2020/obj-sharp.png "Objective Sharpie")

# Working with Objective Sharpie output
The report `Objective Sharpie` provides after running gives you some numbers showing how many various items were wrapped. I saw something like this:
	
	ConstantsInterfaceAssociation (208 instances):
	MethodToProperty (178 instances):
	StronglyTypedNSArray (30 instances):
	PlatformInvoke (6433 instances):
	InferredFromMemberPrefix (185 instances):
	
Additionally, their documentation says to open each both `APIDefinitions.cs` and `StructsAndEnums.cs` and look for a C# attribute called `[Verify]`. `Objective Sharpie` will add that attribute to any method, property or member that it wasn't 100% confident in wrapping, with the expectation that you'll look at it and decide what to do (leave it, change it, remove it.) You then have to remove the `[Verify]` attribute, because the Binder will treat that as a compiler error (thus forcing you to look at them and not just ignore it.)

I was immediately concerned by the output above, because there seemed to be an awful lot of items wrapped for the four header files I intended to bind. I was further concerned when I opened `APIDefinitions.cs` and saw 422 instances of `[Verify]` and the 54,000 lines of code. `StructsAndEnums.cs` was even worse, with 6,646 instances of `[Verify]`. 

After inspecting the files closely, I started noticing that items in the files looked suspiciously like core SDK code.
For example: `public enum NSRoundingMode : ulong` and `public enum NSFileManagerUnmountOptions : ulong`. I began wondering, did `Objective Sharpie` walk the entire header include hierarchy and bind the entire SDK?

After researching this problem further, I discovered an optional flag you can pass `Objective Sharpie` called `-scope [FOLDER]`. This flag limits the headers parsed to only files found in `[FOLDER]`. This sounded like what I wanted, but I had my doubts since this wasn't a default argument. It seemed to good to be true.

Nonetheless, I ran 

	sharpie bind -output=AudioFetch --namespace=AudioFetch --sdk=iphoneos13.2 -scope include/audioFetchSdk include/audioFetchSdk/AudioManager.h

Which is the same as the first attempt, but with a scope that specifies the folder where the AudioFetch headers reside. This time, the output was much more encouraging:

	MethodToProperty (2 instances)

And that was it! It didn't even create a `StructsAndEnums.cs`. And when I opened `APIDefinitions.cs` I saw only 200 lines of code and two instances of `[Verify]`. They were both basically:
	
	// -(BOOL)startDiscovery;
	[Export ("startDiscovery")]
	[Verify (MethodToProperty)]
	bool StartDiscovery { get; }

And, in fact, I DID want this to be a method instead of a property! Well done `Objective Sharpie`!

So this gave me the confidence that I had correctly generated the C# wrapper methods needed to compile the library! I was ready to build the binding.

# Building the Binding 
The first step in setting up the bindings project is to add your Objective-C library `libAudioFetchSDK.a` to the Native References folder in the Visual Studio binding project. According to the documentation, this will cause Visual Studio to automatically create a file called `libAudioFetchSDK.linkwith.cs` where you can configure necessary linker options like other dependency libraries to link in, iOS frameworks, etc.

Well, after adding the Native Reference, I saw no `linkwith` file created. I searched high and low, but nothing turned up. The closest I could find was a new Property Sheet that opened when I right-clicked the Native Reference item `libAudioFetchSDK.a` and chose "Properties". 
![alt text](/images/xaramin-bindings-3-2-2020/prop-page.png "Properties Page")

The properties here looked suspiciously similar to the options that the `linkwith` file supposedly offered, and once again I discovered that the documentation was simply out of date and this Property Sheet was the new method.
This would have been fine, except that some of the options I saw weren't in the documentation for `linkwith` leaving me with some trial and error.

I went ahead and hit compile and to my amazement...it compiled! Time for another simple C# app.

# Building the simple iOS C# App
After setting up a new project and adding the `AudioFetchiOS.dll` reference, I began typing out the code to use it and everything seemed to be working just fine! I got some basic code setup that essentially goes:

	AudioFetch.Shared.Startup( );
	....
	AudioFetch.Shared.StartAudio( );

and hit compile. Moment of truth, right? 
![alt text](/images/xaramin-bindings-3-2-2020/103.png "Errors")

103 errors. Most of them linker errors.

Generally with errors you want to start at the top and work your way down, but scrolling through, I saw a ton similar to this:
	
	MTOUCH: Error MT5210: Native linking failed, undefined symbol: std::logic_error::logic_error(char const*). Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
`std::logic_error` is clearly a std C++ library. OH YEAH! I forgot that in my Objective-C  app, I had added `-lstdc++` to the linker options. I hadn't done any sort of thing in the bindings project.

Translating the documentation for `linkwith` into the Property Sheet I had, it appeared that I could simply use the Linker Flags section. Hopefully it didn't require some proprietary syntax. I added `lstdc++`, compiled, and copied the new DLL over to the prototype app.

Down to 24 errors! It was extremely encouraging to know this was knocking the issues out.
Looking at the new list, the top error said 
	
	/error MT5209: Error: warning: Could not find or use auto-linked framework 'AFNetworking' (AudioFetchCSharp) Native linking
	
Right. AFNetworking. I forgot about that one.

# Building an AFNetworking static library
There were actually several ways to solve the `AFNetworking` dilemma. 
1. Back in my Objective-C app, right-click the CocoaPod in Xcode, choose "Show In Finder", find the `AFNetworking` framework and use that.
2. Look for a pre-built binary online that someone else built and is offering.
3. Go to [AlamoFire's AFNetworking GitHub Repo](https://github.com/AFNetworking/AFNetworking), download their code, and build it myself.

If I'm being honest, I'm too ignorant toward how CocoaPods and Frameworks work to perform option 1, even now. I have too many questions regarding what settings were used when compiling. I ASSUME that it just compiles with the target build settings the project specifies, but given a looming deadline, I considered this a last resort.

Option two sounded good, but realistically I would have had the same concerns regarding "what" was built, and besides that, nobody was offering a binary for `AFNetworking`.

Option three was the most straight forward because I learned earlier how to setup a static library project. It would also be the most useful as I discovered the need for a "Fat Lib" (more on that shortly.)

For anyone coming from a Visual Studio background and switching to Xcode, it can be a lesson in frustration. Xcode uses a lot of different terminology and certain features are not always where you'd expect them to be. Don't get me wrong, it's a mature and useful IDE, it's just very foreign. Figuring out how to add the `AFNetworking` code for compilation into a static library proved to be one of these instances where I spent the majority of the time just trying to understand where things go in Xcode. 

For those wondering, after creating a new Static Library project, click the root Project item in the left-hand workspace, and on the right-hand side choose "Build Phases", Compile Sources, and the little "+" button at the bottom.
![alt text](/images/xaramin-bindings-3-2-2020/xcode-static-lib.png "Static Library")
Then make sure that your static libary's default .m file includes the top-level header for the library.

To then get your compiled file, you would right-click on Products->Library.a and choose "Show in Finder"
Not bad once you know how, right? Now we can link this binary back into our bindings project!

I added the newly built `AFNetworking` lib to my Native References, rebuilt the bindings DLL, and copied it to the C# test app. How many errors would we be down to now?

After compiling, I see we're down to 20 errors. Another improvement! The next couple errors are both along the lines of

	MTOUCH: Error MT5210: Native linking failed, undefined symbol: _AudioComponentFindNext. Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
	MTOUCH: Error MT5210: Native linking failed, undefined symbol: _MPMusicPlayerControllerVolumeDidChangeNotification. Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
Googling these symbols tells us that they reside in the `AudioToolbox` and `MediaPlayer` frameworks respectively. Ok, so I need to link in the frameworks. No problem!

# Linking Frameworks
Back in the bindings project, I didn't see an obvious way to say "Include these frameworks." You can't add them as Native References, and there are no linker options at the Project level. The best I could find was the "Frameworks" field in the Property Page for the Native Reference of `libAudioFetch.a`. 

Comparing this with the documentation for that non-existent `linkwith` file, it seemed clear that this was where you could put dependency frameworks. Other's online questions / answers also lead me to believe this would be where I'd add it.

I had questions around how to specify the frameworks, but the answer I finally found was "A space delimitted list of frameworks without the extension") 

![alt text](/images/xaramin-bindings-3-2-2020/frameworks.png "Static Library")

Just like this.

# Are we there, yet?
Back to the prototype project, I compiled yet again, and only five errors!
The next error up was 

	/MTOUCH: Error MT5210: Native linking failed, undefined symbol: _res_9_init. Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
At this point it's pretty clear that I'm missing another library. Googling `res_9_init` (always remove the _ prefix) tells us it lives in `libresolv`. And wait...waaaay back when we started the sample Objective-C app, I did have to add a link command for `libresolv.tbd`. This feels familiar. But what's a `tbd` file? Is it safe to simply link to the "regular" `libresolv` library?

After searching for the `tbd` extension in relation to Xcode, I saw that it's a ["Text-based Definition File"](https://fileinfo.com/extension/tbd) used by Xcode to make the Xcode SDK download smaller. The final product linked in is indeed the normal `libresolv` library. So we're safe to include it.

Since it's not a framework, and we're not using an IDE specific method (like the `.tbd`) I can link it in like any other system library. 

Linker flags becomes:
	
	-lstdc++ -lresolv

and finally, at long last, the simple C# test app compiles! 

Unless you want to build for the iPhone Simulator.

# Are you kidding me? (x86, Arm and "fat libs")
The errors looked very similar to the linker errors I had already solved. So how could I be getting link errors again?
Scrolling through the log, intermixed with the errors were a couple of warnings

	ld: warning: ignoring file /obj/iPhoneSimulator/Debug/device-builds/iphone 11-13.3/mtouch-cache/libAFNetworking.a, building for iOS Simulator-x86_64 but attempting to link with file built for iOS-arm64
	
	ld: warning: ignoring file /obj/iPhoneSimulator/Debug/device-builds/iphone 11-13.3/mtouch-cache/libAudioFetch.a, building for iOS Simulator-x86_64 but attempting to link with file built for iOS-arm64 
	
What we can see here is the linker saying "Hey, the iPhone Simulator is built for x86 architectures. So I'm going to ignore these incompatible Arm libraries."

The thing about that is it leaves the iPhone Simulator with no compatible `AudioFetch` or `AFNetworking` libraries to link in, which causes us to end up with link errors for the missing symbols. What to do?

There are basically three ways to solve this.
1. Include both x86 and Arm libraries in the bindings project.
This is straight forward enough, but it would cause the above warning to show anytime I built, because for whatever architecture is targeted, the linker would ignore the libraries for the OTHER architecture.

2. Setup different project configurations
This is my least favorite option, since we'd have to create four project configurations (x86 Debug, x86 Release, Arm Debug, Arm Release).

3. Create a "fat lib".
A "fat lib" (as I learned making these bindings) is a single library with both architectures within it. Because they exist as "slices" within the library, there won't be any warnings when building because the linker will only grab the "slice" for the active architecture. The only downside to this is a double-sized static library, since the single library now contains both architectures. This ends up ok being though, because the final executable only links in the architecture it's compatible with, ensureing our final app size isn't bloated with unused code.

# Creating a fat lib
When I started researching how to make a fat lib, I ran into several methods and preferred approaches. I ended up finding a simple `Makefile` in, of all places, that out-dated [iOS Bindings Walkthrough.](https://docs.microsoft.com/en-us/xamarin/ios/platform/binding-objective-c/walkthrough?tabs=macos)

You setup an Xcode Static Library project and include both library architectures in the “Link Binary with Libraries” Build Phase. Then in the parent folder, you simply run the script. Here's the actual one I used

	XBUILD=/Applications/Xcode.app/Contents/Developer/usr/bin/xcodebuild
	PROJECT_ROOT=./Release
	PROJECT=$(PROJECT_ROOT)/AudioFetchFat.xcodeproj
	TARGET=AudioFetchFat

	all: lib$(TARGET).a

	lib$(TARGET)-i386.a:
		$(XBUILD) -project $(PROJECT) -target $(TARGET) -sdk iphonesimulator -configuration Release clean build
		-mv $(PROJECT_ROOT)/build/Release-iphonesimulator/lib$(TARGET).a $@
	
	lib$(TARGET)-arm64.a:
		$(XBUILD) -project $(PROJECT) -target $(TARGET) -sdk iphoneos -arch arm64 -configuration Release clean build
		-mv $(PROJECT_ROOT)/build/Release-iphoneos/lib$(TARGET).a $@

	lib$(TARGET).a: lib$(TARGET)-i386.a lib$(TARGET)-arm64.a
		xcrun -sdk iphoneos lipo -create -output $@ $^

	clean:
		-rm -f *.a *.dll

Note that I didn't care about Armv7 (32-bit) so I omitted it.

# Finally
With a fat lib created for both `AudioFetch` and `AFNetworking`, I copied them into the bindings project and rebuilt the DLL. Sure enough, `AudioFetchiOS.dll` went from about 9.8 megs to 18.7. 

At long last I copied this DLL into the C# test app and hit compile for the iOS Simulator. No warnings, no errors. I switched to the hardware Device and hit compile. No warnings, no errors.

I launched the app, waited, and before long I saw the AudioFetch output telling me it had found a stream and connected! iOS bindings were done.

## Summary
I've been writing code for 18 years, and this was easily one of the most technically daunting tasks I've had to do. It was also one of the most rewarding!

My purpose in documenting my experience was to help solidify my own understanding of all the various technologies I worked with (and sure enough, I had to go back and validate some of the assumptions I had made.) and, more importantly, provide a resource for others that are trying to create bindings.

I found during this process that bindings are common enough that you'll eventually need one, but not so common that it's well documented in a single place. Even this piece serves as just another repository of information, and not really an exhaustive overview of how to generate them.

Please don't hesitate to reach out if you have any comments, questions, or corrections. There's still so much I don't know, and probably much more efficient ways to do a lot of what I'm doing.