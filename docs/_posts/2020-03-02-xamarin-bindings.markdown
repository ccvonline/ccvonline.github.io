---
layout: post
title:  "Xamarin Bindings"
date:   2020-03-02 10:29:05 -0700
categories: xamarin bindings c# ios android
---
# The Project
I work as a software engineer at a church and we have a Mobile App for both Android and iOS, written in C# on the Xamarin platform. We have a growing population of native Spanish speakers, and decided we wanted to offer real time Spanish translation to our members, integrated into the Mobile App.

In production a translator would be speaking into a device that broadcasts on our public Wifi network. Users could then launch our Mobile App, connect to our Wifi and listen to the stream, all in real time.

After R&Ding several devices, we settled on one called AudioFetch, because the hardware was reliable and they offered an SDK in the form of native iOS and Android libraries you link in and use. Perfect!

So this project began with two files and a straight forward task. Take `libAudioFetchSDK.a` and `afaudiolib.aar` and come out with `AudioFetchiOS.dll` and `AudioFetchDroid.dll`. On paper, this simply requires using Visual Studio to create C# bindings for both iOS and Android respectively. They already have the project templates for this, so what could be simpler?

To start, I wanted to gain an understanding of how these libs worked in their native environments. This was a very important first step, because I anticipated there would be problems just in generating the bindings themselves, and if I wasn't intimately familiar with the libraries, it would be extremely difficult to separate an issue with the native library with an issue in my binding work.

I flipped a coin and it landed on Android, so down the rabbit hole I went...

# Android Bindings

## Building a simple Android App
Step 1: Download Android Studio and build a simple Java app that uses `afaudiolib.aar` to stream audio.
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

Notice they're using `RXJava` as a messaging system for their library. This is going to be important.

After getting an example app working and understanding how to use their library, it was time to generate C# bindings.

## Generating C# Bindings
The first step was to create a new Android Bindings project in Visual Studio.
[PIC]

I added `afaudiolib.aar` and set its build action to `LibraryProjectZip`, since this was the main file I wanted bindings generated for.

### Missing dependency libs
Next, I needed to add the jar files that `afaudiolib.aar` depends on, which were listed in the gradle config above. 
But wait...the only place I knew to look for libraries was the `libs` folder in my project.

In that `libs` folder, the only file I had was `afaudiolib.aar`. So where are the other four files I need?

Notice the definition for the depencies for the missing libs:

    implementation 'io.reactivex.rxjava2:rxjava:2.2.0'
    
That instructs gradle to go download the lib AT BUILD TIME and link it in. Which means there's no physical file for me to go get. Hrm. I need those libs.

After scouring the internet, I came across this extremely helpful gradle action:
https://gist.github.com/n-belokopytov/d44949590748b096c1a497008b761d04

This is a gradle dependency copier, which will take the downloaded files, resolve their names, and store them in a folder on the filesystem!

After figuring out how to use gradle at command line, I typed `gradlew copyDependenciesDebug` and got the following output:

    // Copying dependency:
    // /Users/jered/.gradle/caches/modules-2/files-2.1/io.reactivex.rxjava2/rxjava/2.2.0/2c622e8f1cab8d5871d32bb2023af515216056d8/rxjava-2.2.0.jar

Yay! I went to each folder and grabbed my four dependencies.

Now, before jumping right back into the bindings project, I took a detour and reconfigured my prototype Java app to use these copied libs instead of the download-at-build-time versions.
There are so many unknowns when generating bindings it's best to go slow and validate every step.

My libs folder now looked like this
![alt text](https://github.com/ccvonline/ccvonline.github.io/docs/images/a.png "Image Hover")

and I changed the dependency includes to use local jars instead of downloading ones, like so:

    implementation 'io.reactivex.rxjava2:rxjava:2.2.0' -> implementation(name: 'rxrelay-2.0.0', ext: 'jar')

I then built and ran the app, and it worked as before, but now with all libs stored locally!
NOW I had everything I needed to resume generating the bindings.

### Binding Compiler Errors
I added the four additional libs to the Jar folder, and because they needed to actually be bundled into the DLL, set their build action to `EmbeddedReferenceJar`
[PIC]

Next I hit rebuild, and waited. 5 errors.
[PIC]

One error was an illegal naming convention in C#, where a class member cannot be the same name as its owning class.

    Error CS0542: 'Channel': member names cannot be the same as their enclosing type (CS0542)

The other errors were all a missing interface implementation, along the lines of:

    Error CS0535: 'Apb' does not implement interface member 'IComparable.CompareTo(Object)' (CS0535)

The fix for issues like these is to use the `Metadata.xml` file and implement transforms that give the binder extra instructions to fix these issues.

To fix the first issue, we just tell the binder to use a different name for the C# member.

    <attr path="/api/package[@name='com.audiofetch.afaudiolib.dal']/class[@name='Channel']/field[@name='channel']" name="managedName">ChannelIndex</attr>

However, the first attempt at doing this failed with a runtime (not compile) error, because instead of using `name="managedName"`, I had `name="name"`. 

I got a runtime exception whenever I referenced `ChannelIndex` with a message saying that it was a non-existent property in Java. What I failed to realize was `name="name"` told the binder to generate a C# wrapper called `ChannelIndex` bound to a Java member ALSO called `ChannelIndex`, like so:

		[Register ("ChannelIndex")]
		public int ChannelIndex {
			get {
				const string __id = "ChannelIndex.I";

				var __v = _members.InstanceFields.GetInt32Value (__id, this);
				return __v;
			}
			set {
				const string __id = "ChannelIndex.I";

				try {
					_members.InstanceFields.SetValue (__id, this, value);
				} finally {
				}
			}
		}
The problem is `ChannelIndex` doesn't exist in Java. In Java it's just called `channel`.

What I needed was for the C# WRAPPER to be `ChannelIndex`, but have it lookup the underlying Java property by `channel`.
By using `name="managedName"` we told the binder exactly that, and ended up with:

        [Register ("channel")]
		public int ChannelIndex {
			get {
				const string __id = "channel.I";

				var __v = _members.InstanceFields.GetInt32Value (__id, this);
				return __v;
			}
			set {
				const string __id = "channel.I";

				try {
					_members.InstanceFields.SetValue (__id, this, value);
				} finally {
				}
			}
		}
and it worked!

For the missing interface implementations, these were a little more straight forward. We simply needed to instruct the binder to generate bindings for the missing types.

    <add-node path="/api/package[@name='com.audiofetch.afaudiolib.dal']/class[@name='Apb']">
        <method name="compareTo" return="int" abstract="false" native="false" synchronized="false" static="false" final="false" deprecated="not deprecated" visibility="public">
            <parameter name="p0" type="java.lang.Object" />
        </method>
        
        <method name="compare" return="int" abstract="false" native="false" synchronized="false" static="false" final="false" deprecated="not deprecated" visibility="public">
            <parameter name="p0" type="java.lang.Object" />
            <parameter name="p1" type="java.lang.Object" />
        </method>
    </add-node>

Now the binding compiled! However, it still wasn't time to move this into our production Mobile App. My next step was to build a C# prototype app that would use this DLL in order to validate that the bindings were working correctly on their own.

### Java compiler errors
After setting up a basic app and adding the library as a reference, I compiled. And error.

    Error JAVAC0000:  error: OnReportCompletedListener is not public in ReportController; cannot be accessed from outside package com.audiofetch.afaudiolib.bll.colleagues.ReportController.OnReportCompletedListener

After doing some research, I understood the problem to be an access modifier issue with the way `IReportCompletedListener` was being generated. I spent a little time attempting to transform its access to be public to no avail, and finally realized it wasn't necessary to transform, because I had no need of referencing it in my DLL.

So my "fix" was the tell the binder not to generate bindings for this class.

    <remove-node path="/api/package[@name='com.audiofetch.afaudiolib.bll.colleagues']/class[@name='ReportController']" />
As a side note, I'd love feedback on how to correctly fix this in the future.

So now we have a fully compiling C# prototype app using the bindings. 

## Building the C# Prototype App
It was time to begin implementing functionality that used the DLL.
It didn't take long for the next issue to become apparent.

### Dependency libraries surfaced in C#
Note earlier that I said sample usage of the AudioFetch service looked like this:

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

As I began to type the C# code to use the AudioFetch DLL, I ran into an issue.

    AFAudioService.Api(). //Yay!
                OutMsgs(). //Ok!
                AsFlowable(). //This didn't exist...

The problem was `AsFlowable()` didn't have C# methods. Why not? Because they weren't bound.

What we bound was the highest level API for AudioFetch. And while yes, we did link in the java libs that AudioFetch needed, which allows IT to compile, its method `OutMsgs()` returns a straight RxJava object, which means it needed exposure to C# as well.

Before I even considered building bindings for these myself, I checked nuget, and thankfully ran across packages that ALREADY BOUND THESE. 
Thank you NAXAM! https://github.com/naxam

Now, I could have just added these packages to my C# app, but the problem with that is `AsFlowable()` is a method on the object returned by `OutMsgs()`. And `AsFlowable()` was never bound because at BINDING TIME the binder wasn't aware of the java type (because the jar was being included as a bundled resource and not a reference for binding.)

So if I had just included the packages in the prototype app, I basically would have been able to do:

    AFAudioService.Api(). //Yes
                OutMsgs(). //Yes
                AsFlowable(). //No, because it was never bound.
                .ObserveOn //Yes, because it was part of RxJava

Which obviously won't work. Back to the bindings project.

Using nuget I added these IN THE BINDINGS PROJECT. Since these bindings obviously include the jar files, I also removed those specific jar files from my Jar references. (Otherwise we'll get duplicate symbol linker errors)
This left me with three Jar files (`afaudiolib.aar`, `guava-19.0.jar`, and `rxrelay-2.0.0.jar`) and two C# references (`Naxam.RxAndroid.Droid` and `Naxam.RxJava2.Droid`)
[PIC]

Successful compile! And as I hoped, the binder saw the C# binding references and was able to generate the missing C# bindings for the lower level AudioFetch API (`AsFlowable()`) in addition to providing me with the DLLs for RXJava and RXAndroid.

## Bringing it all together
Now back in the C# Prototype App, I was able to add references to `RXAndroid` and `RXJava`, and well as `AudioFetchDroid`.

After cleaning the solution, I began typing, and was able to type this:

    AFAudioService.Api().
        OutMsgs().
        AsFlowable().
        ObserveOn(IO.Reactivex.Android.Schedulers.AndroidSchedulers.MainThread())
        .Subscribe(this);

It compiled, it linked, and it ran! Finally I had successfully created C# bindings for AudioFetch and its dependency libraries, and was able to use it in a prototype app.

A this point, with it working entirely in C#, I felt confident enough to move these DLLs into our production app, and we are now sucessfully streaming audio from AudioFetch 
in our app!

Next up, iOS.

# iOS Bindings

## Building a simple iOS app
I began with the same process that proved useful for Android, and went about building a simple Objective-C prototype app that used the `libAudioFetchSDK.a` library.

Similar to Android, they have some basic setup documentation, and it says:
1. Setup a new XCode Project
2. Add the AudioFetch SDK headers
3. Add the following libraries to the link Build Phase: `libAudioFetchSDK.a`, `MediaPlayer.framework`, `libresolv.tbd`
4. Add `lstdc++` to the linker flags in Build Settings. (Basically include the C++ standard library)
5. Use CocoaPods to include the dependency library `AFNetworking`.

Step five was confusing for me on multiple levels. First, I'd never used CocoaPods before. I assumed it was a dependency manager for XCode? (It is.) and I assumed that even though AFNetworking started with AF, like "AudioFetch" abbreviates to, this was just an unfortunate and extremely confusing coincidence. (Yup.)

After getting a crash course in setting up and using CocoaPods, I had my prototype app building. 

Usage looked pretty straight forward, with calls like `[[AudioManager.sharedInstance] startAudio];` 

As I put together basic playback code, I was mindful of watching out for any API that surfaced AFNetworking, and thankfully didn't run into any. So unlike Android, it appeared that AFNetworking was used only internally by their library, and I wouldn't have to worry about generating bindings for this. As a bonus, if I do want to utilize any AFNetworking in my code, I did notice that once again NAXAM comes to the rescue with an AFNetworking C# binding.

So! Their example app is building. It's time to generate some C# bindings.

## Generating C# iOS bindings
A quick Google search landed me here: https://docs.microsoft.com/en-us/xamarin/ios/platform/binding-objective-c/
and it's never an encouraging sign when the front page has a one hour YouTube tutorial. Especially one filmed in 2015.

Because I had success going step by step with Android, I wanted to do the same here, so I watched the video. Most of it was either too low level for what I needed (binding C libraries) or review of Xamarin basics, but it was worth it purely for giving me a review of the concepts and various pieces that I'd need. It helped show me what unknowns I would need to account for.

So step 1 was to create a Xamarin iOS Bindings project, just like Android. This was encouraging! Unlike Android, there were no XML files. Instead I had an `APIDefinitions.cs` and `StructsAndEnums.cs`. According to their documentation, I would use a tool called `objective sharpie` to parse the AudioFetch headers, and that would output an `APIDefinitions.cs` and `StructsAndEnums.cs` that contained most of the bindings for me. Basically it autogenerates the bindings, and then leaves it to me to fix any issues that come up. So similar to how Android parses the libraries and lets me make adjustments in `Metadata.xml`.

### Trying the "Bindings Complete Walkthrough"
The documentation for Objective Sharpie sounded complex enough (including a literal warning at the top of the page that this tool was only to be used by "advanced" developers) that I decided to use their Walkthrough called "Binding an iOS Objective-C Library" that takes you step by step.

This proved useful because it taught me how to generate a static library in XCode, which was something I would end up needing later for AFNetworking.

Unfortunately when attempting to build the static library, it became apparent that this walkthrough was about 4 years out of date, and I wasn't going to be able to proceed.

So I decided to throw caution to the wind and just try and run Objective-Sharpie against the AudioFetch headers.

Usage for Objective Sharpie said to go to the folder containing my headers and run a simple command:
	
	sharpie bind -output=AudioFetch --namespace=AudioFetch --sdk=iphoneos13.2 include/audioFetchSdk/AudioManager.h

Running this resulted in: `zsh: command not found: sharpie` What? Back to the documentation.
Oh. This tool isn't included as part of the Visual Studio Mac / Xamarin installation. It's a separate download linked within the documentation for Objective Sharpie. 

After downloading it, I ran `sharpie update` to verify I was up-to-date (per their documentation) and it said I was! Now we're ready!

I again ran the sharpie command above, and started to see output! Unfortunately that output included an error stating it couldn't generate the APIDefinitions, due to something related to `macOS Catalyst`.

I recognized Catalyst as the name of Apple's project for bringing iPad apps to macOS, and immediately wondered if this supposedly up-to-date version of sharpie was actually the most up-to-date. Sure enough, after doing some additional searching, I discovered I had downloaded version 3.4, and there existed 3.5.22. It turns out they changed the update channel that newer versions update with, so I had to find an alternative download link for a version newer than 3.4, and then update THAT version to get to the latest 3.5.22.

Now with the latest version installed, I ran the sharpie command again, and saw a successful `APIDefinitions.cs` and `StructsAndEnums.cs` generated! Success?

### Working with Objective-Sharpie output
The report Objective-Sharpie provides after running gives you some numbers showing how many various items were wrapped. I saw something like this:
	
	ConstantsInterfaceAssociation (208 instances):
	MethodToProperty (178 instances):
	StronglyTypedNSArray (30 instances):
	PlatformInvoke (6433 instances):
	InferredFromMemberPrefix (185 instances):
	
Additionally, their documentation says to open each file, `APIDefinitions.cs` and `StructsAndEnums.cs` and look for a C# attribute called `[Verify]`. Objective-Sharpie will add that attribute to any method, property or member that it wasn't 100% confident in wrapping, with the expectation that you'll look at it and decide what to do (leave it, change it, remove it.) You then have to remove the `[Verify]` attribute, because the Binder will treat that as a compiler error (thus forcing you to look at them and not just ignore it.)

I was immediately concerned by the output above, because there seemed to be an awful lot of items wrapped for the four header files I intended to bind. I was further concerned when I opened `APIDefinitions.cs` and saw 422 instances of `[Verify]` and the 54,000 lines of code. `StructsAndEnums.cs` was even worse, with 6,646 instances of `[Verify]`. 

After inspecting the files closely, I started noticing that items in the files looked suspiciously like core SDK code.
For example: `public enum NSRoundingMode : ulong` and `public enum NSFileManagerUnmountOptions : ulong`. I began wondering, did Objective-Sharpie walk the entire Header dependency tree and bind the entire SDK?

After researching this problem further, I discovered an optional flag you can pass Objective-Sharpie called `-scope [FOLDER]`. This flag limits the Header walking to only files found in `[FOLDER]`. This sounded like what I wanted, but I had my doubts since this wasn't a default argument. It seemed to good to be true.

Nonetheless, I ran 

	sharpie bind -output=AudioFetch --namespace=AudioFetch --sdk=iphoneos13.2 -scope include/audioFetchSdk include/audioFetchSdk/AudioManager.h

Which is the same as the first attempt, but with a scope that specifies the folder where the AudioFetch headers reside. This time, the output was much more encouraging:

	MethodToProperty (2 instances)

And that was it! It didn't even create a `StructsAndEnums.cs`. And when I opened `APIDefinitions.cs` I saw only 200 lines of code and two instances of `[Verify]`. They were both basically:
	
	// -(BOOL)startDiscovery;
	[Export ("startDiscovery")]
	[Verify (MethodToProperty)]
	bool StartDiscovery { get; }

And, in fact, I DID want this to be a method instead of a property! Well done Objective-Sharpie!

So this gave me the confidence that I had correctly generated the C# wrapper methods needed to compile the library! I was ready to build the binding.

### Building the Binding 
The first step in setting up the bindings project is to add your Objective-C library `libAudioFetchSDK.a` to the Native References folder in the project. According to the documentation, this will cause Visual Studio to automatically create a file called `libAudioFetchSDK.linkwith.cs` where you can configure necessary linker options like other dependency libraries to link in, iOS frameworks, etc.

Well, after adding the Native Reference, I saw no `linkwith` file created. I searched high and low, but nothing turned up. The closest I could find was a new Property Sheet that opened when I right-clicked the Native Reference item `libAudioFetchSDK.a` and chose "Properties". 
[PIC]
The properties here looked suspiciously similar to the options that the `linkwith` file supposedly offered, and once again I discovered that the documentation was simply out of date and this Property Sheet was the new method.
This would have been fine, except that some of the options I saw weren't in the documentation for `linkwith` leaving me with some trial and error.

I went ahead and hit compile and to my amazement...it compiled! Time for another C# prototype app.

### iOS CSharp Prototype App
After setting up a new project and adding the `AudioFetchiOS.dll` reference, I began typing out the code to use it and everything seemed to be working just fine! I got some basic code setup that essentially goes:

	AudioFetch.Shared.Startup( );
	....
	AudioFetch.Shared.StartAudio( );

and hit compile. Moment of truth, right? 
[PIC]

103 errors. Most of them linker errors.

Generally with errors you want to start at the top and work your way down, but scrolling through, I saw a ton similar to this:
	
	MTOUCH: Error MT5210: Native linking failed, undefined symbol: std::logic_error::logic_error(char const*). Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
`std::logic_error` is clearly a std C++ library. OH YEAH! I forgot that in my Objective-C prototype app, I had added `-lstdc++` to the linker options. I hadn't done any sort of thing in the Bindings Project.

Translating the documentation for `linkwith` into the Property Sheet I had, it appeared that I could simply use the Linker Flags section. Hopefully it didn't require some proprietary syntax. I added `lstdc++`, compiled, and copied the new DLL over to the prototype app.
[PIC] 

Down to 24 errors! It was extremely encouraging to know this was knocking the issues out.
Looking at the new list, the top error said 
	
	/error MT5209: Error: warning: Could not find or use auto-linked framework 'AFNetworking' (AudioFetchCSharp) Native linking
	
Right. AFNetworking. I forgot about that one.

### Building an AFNetworking static library
There were actually several ways to solve the AFNetworking dilemma. 
1. Right-click the CocoaPod in Xcode, choose "Show In Finder", find the AFNetworking framework and build that.
2. Look for a pre-built binary online that someone else built and is offering.
3. https://github.com/AFNetworking/AFNetworking download their code (for the version AudioFetch used) and build it myself.

If I'm being honest, I'm too ignorant toward how CocoaPods and Frameworks work to perform option 1, even now. I have too many questions regarding what settings were used when compiling. I ASSUME that it just compiles with the target build settings the project specifies, but given a looming deadline, I considered this a last resort.

Option two sounded good, but realistically I would have had the same concerns regarding "what" was built, and besides that, nobody was offering a binary for AFNetworking.

Option three was the most straight forward--and because I setup the static library project--removed any unknown variables from the equation. It would also be the most useful as I discovered the need for a "Fat Lib" (more on that shortly.)

For anyone coming from a Visual Studio background and switching to Xcode, it can be a lesson in frustration. Xcode uses a lot of different terminology and certain features are not always where you'd expect them to be. Don't get me wrong, it's a mature and useful IDE, it's just very foreign. Figuring out how to add the AFNetworking code for compilation into a static library proved to be one of these instances where I spent the majority of the time just trying to understand where things go in Xcode. 

For those wondering, after creating a new Static Library project, click the root Project item in the left-hand workspace, and on the right-hand side choose "Build Phases", Compile Sources, and the little "+" button at the bottom.
[PIC]
Then make sure that your .m file includes the top-level header for the library.

To then GET your library, you would then right-click on Products->Library.a and choose "Show in Finder"
[PIC]

Not bad once you know how, right? Now we can link this binary back into our bindings project!

I added the newly built `AFNetworking` lib to my Native References, rebuild the bindings DLL, and copied it to the prototype app. How many errors would we be down to now?

After compiling, I see we're down to 20 errors. Another improvement! The next couple errors are both along the lines of

	MTOUCH: Error MT5210: Native linking failed, undefined symbol: _AudioComponentFindNext. Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
	MTOUCH: Error MT5210: Native linking failed, undefined symbol: _MPMusicPlayerControllerVolumeDidChangeNotification. Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
Googling these symbols tells us that they reside in the `AudioToolbox` and `MediaPlayer` frameworks respectively. Ok, so I need to link in the frameworks. No problem!

### Linking Frameworks
Back in the bindings project, I didn't see an obvious way to say "Include these frameworks." You can't add them as Native References, and there are no linker options at the Project level. The best I could find was the "Frameworks" field in the Property Page for the Native Reference to `libAudioFetch.a`. 

Comparing this with the documentation for that non-existent `linkwith` file, it seemed clear that this was where you could put dependency frameworks. Other's online questions / answers also lead me to believe this would be where I'd add it.

I had questions around how to specify the frameworks, but the answer I finally found was "A space delimitted list of frameworks without the extension") 

Just like this.
[PIC]

### Are we there, yet?
Back to the prototype project, I compiled yet again, and only five errors!
The next error up was 

	/MTOUCH: Error MT5210: Native linking failed, undefined symbol: _res_9_init. Please verify that all the necessary frameworks have been referenced and native libraries are properly linked in. (MT5210)
	
At this point it's pretty clear that I'm missing another library. Googling `res_9_init` (always remove the _ prefix) tells us it lives in `libresolv`. And wait...waaaay back when we started the prototype Objective-C app, I did have to add a link command for `libresolv.tbd`. This feels familiar. But what's a `tbd` file? Is it safe to simply link to the "regular" libresolv library?

After searching for the `tbd` extension in relation to Xcode, we see that it's a "text-based stub library" used by Xcode to make the Xcode SDK download smaller. The final product linked in is indeed the normal `libresolv` library. So we're safe to include it.

Since it's not a framework, and we're not using an IDE specific method (like the `.tbd`) we can link it in like any other system library. 

Linker flags becomes:
	
	-lstdc++ -lresolv

and finally, at long last, the C Sharp Prototype app compiles! 

Unless you want to build for the iPhone Simulator.

### Are you kidding me? (x86, Arm and "fat libs")
The errors looked very similar to the linker errors we had already solved. So how could we be getting link errors again?
Scrolling through the log, intermixed with the errors were a couple of warnings

	ld: warning: ignoring file /obj/iPhoneSimulator/Debug/device-builds/iphone 11-13.3/mtouch-cache/libAFNetworking.a, building for iOS Simulator-x86_64 but attempting to link with file built for iOS-arm64
	
	ld: warning: ignoring file /obj/iPhoneSimulator/Debug/device-builds/iphone 11-13.3/mtouch-cache/libAudioFetch.a, building for iOS Simulator-x86_64 but attempting to link with file built for iOS-arm64 
	
What we can see here is the linker saying "Hey, the iPhone Simulator is built for x86 architectures. So I'm going to ignore these incompatible Arm libraries."

The thing about that is it leaves the iPhone Simulator with no compatible AudioFetch or AFNetworking libraries to link in, which causes us to end up with link errors for the missing symbols. What to do?

There are basically three ways to solve this.
1. Include both x86 and Arm libraries in the bindings project.
This would work, and be straight forward enough, but it cause the above warning to show anytime we built, because whatever it would let us know it's ignoring whatever architecture we weren't building for.

2. Setup different project configurations
This is my least favorite option, since we'd have to create four project configurations (x86 Debug, x86 Release, Arm Debug, Arm Release).

3. Create a "fat lib".
A "fat lib" (as I learned making these bindings) is a single library with both architectures within it. Because they exist as "slices" within the library, there won't be any warnings when building because the linker will only grab the "slice" for the active architecture. The only downside to this is a double-sized static library, since the single library now contains both architectures. This ends up ok being though, because the final executable only links in the architecture it's compatible with, ensureing our final app size isn't bloated with unused code.

### Creating a fat lib
When I started researching how to make a fat lib, I ran into several methods and preferred aproaches. I ended up finding a simple Makefile in, of all places, the out-dated iOS Bindings walkthrough here: https://docs.microsoft.com/en-us/xamarin/ios/platform/binding-objective-c/walkthrough?tabs=macos

You setup an Xcode Static Library project and include both library architectures in the Link With Build Phase. Then in the parent folder, you simply run the script. Here's the actual one I used

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

Note that I don't care about Armv7 so I'm omitting it.

### Finally
With a fat lib created for both AudioFetch and AFNetworking, I copied them into my bindings project, and rebuilt the DLL. Sure enough, AudioFetchiOS.Dll went from about 9.8 megs to 18.7. 

At long last I copied this DLL into the C# Prototype App and hit compile for Simulator. No warnings, no errors. I switched to the hardware Device and hit compile. No warnings, no errors.

I launched the app, waited, and before long I saw the AudioFetch output telling me it had found a stream and connected! iOS bindings were done.

# Summary
I've been writing code for 18 years, and this was easily one of the most technically daunting tasks I've had to do. It was also one of the most rewarding!

My purpose in writing this experience out was to help solidify my own understanding of all the various technologies I worked with (and sure enough, I had to go back and validate some of the assumptions I had made.) and, more importantly, provide a resource for others that are trying to create bindings.

I found during this process that bindings are common enough that you'll eventually need one, but not so common that it's well documented in a single place. Even this piece serves as just another repository of information, and not really an exhaustive overview of how to generate them.

Please don't hesitate to reach out if you have any comments, questions, or corrections. There's still so much I don't know, and probably much more efficient ways to do a lot of what I'm doing.