---
description: A story of a young developer and Tor
icon: shovel
---

# Developer Story: Arti Integration Journey

For my first official independent assignment at Guardian Project, I have been tasked with fleshing out a sample app to demonstrate how Arti (Tor in Rust) can be integrated into an app.

While Arti can be imported as a binary library, the new codebase and build systems makes it easier for anyone to compile and include directly from the source code. As such, like many other developers do for their projects, I built it locally. For this, I had to install the Rust environment with the instructions [here](https://www.rust-lang.org/tools/install). After that, and installing [Android Studio](https://developer.android.com/studio/install?gad\_source=1\&gclid=CjwKCAjwp4m0BhBAEiwAsdc4aJZZxVIVQkZdI-HK89uKQw1ClXRhcwpkEL1wk7oZYZwGqES5ZM93XxoCYEoQAvD\_BwE\&gclsrc=aw.ds), I got to work building the Arti binary.

The first step for that is to clone the Arti Mobile (Experimental) code [repository](https://gitlab.com/guardianproject/tormobile/arti-mobile-ex) on Gitlab, also known as “arti-mobile-ex”. As it will tell you in the readme, from your terminal, you will first need to navigate to the common folder in the repo and run “make android”. This will run the android build in the [Makefile](https://opensource.com/article/18/8/what-how-makefile), which you can find and look through in the common folder. If you are on a Mac, like myself, you will likely get some warnings from your computer about unverified libraries used in the project, which will prevent you from building properly. As a result, you may have to re-run it a couple of times after navigating to System Settings > Privacy and Security and scrolling down and choosing to [allow the packages to be installed](https://support.apple.com/en-tj/guide/mac-help/mh40616/mac) anyways. You may be (reasonably) apprehensive to do so, but in big projects that involve a lot of different packages, there will likely be one or two that come from an “unidentified developer” by Apple. This is usually no cause for panic, and you can always do a bit of research on the package to put your mind at ease. Then, you will need to open Android Studio, and open the folder arti-mobile-ex > android > sample as a project.

The sample app, in its current state, has only one functionality. If you press a button in the bottom left corner, it will first attempt to connect to Tor with Arti. If that fails, then it will attempt to connect to Tor directly. If neither of those work, it will show an error message. It will display the connection status, and if you press the refresh button again, it will re-connect.

The first task I am taking on is giving the user the option to connect to Tor directly, via Snowflake, or Lyrebird/Obfs4. If you are not sure what any of those are, neither was I! These connection options are all types of [Pluggable Transports](https://www.pluggabletransports.info/): tools that developers use to [disguise](https://support.torproject.org/glossary/pluggable-transports/) their users’ traffic. Many countries in which users might want to connect to the Tor network block direct connections to the Tor network. As such, developers use pluggable transports to make that traffic look like something else.

Tor itself mainly uses the “obfsuscating” line of pluggable transports, the most recent version of which is obfs4 (also called Lyrebird). Obfs4, which is built into the Tor Browser, makes Tor traffic look [random](https://tb-manual.torproject.org/circumvention/). [Snowflake](https://snowflake.torproject.org/) is another type of pluggable transport which [functions](https://www.pluggabletransports.info/how-transports/) in a more peer-to-peer way, using many temporary connections.

UI-wise, I thought the best way to implement this feature would be through a dropdown menu, which would have to communicate with the App.java. This is because the Arti Proxy had to be run from the context of App.java. After some (slightly arduous) searching, I realized that I could easily just call functions from App.java using (App)getApplication()) like so:

`((App)getApplication()).connectTorDirect();`

So, in my App.java, I created three methods: connectTorDirect(), connectWithLyrebird(), and connectWithSnowflake() that took the necessary arguments needed to start Arti with each of the respective options.

Then, going back to my MainActivity, I worked out a UI flow for the app on the whole. First, the user would select the type of PT they wanted, causing the necessary input fields to appear.

At the bottom of the screen, I coded up start, stop, and reload buttons. By default, these buttons are all disabled. But once the user selects a valid PT type, the start button becomes enabled. After the user then correctly fills out the input fields, they can click the start button and thus enable the stop and reload buttons. Some time down the line, I added regex to the bridge lines to prevent the user from giving invalid input. In Java, I learned that this can be done using the [Pattern](https://docs.oracle.com/javase/8/docs/api/java/util/regex/Pattern.html) and [Matcher](https://docs.oracle.com/javase/8/docs/api/java/util/regex/Matcher.html) classes.



Some XML markup code:

```xml
<Button android:id="@+id/startButton"
	android:layout_width="90dp"
	android:layout_height="60dp"
	android:layout_marginStart="32dp"
	android:layout_marginEnd="44dp"
	android:layout_marginBottom="60dp"
	android:backgroundTint="@color/teal_200"
	android:text="@string/start"
	android:textColor="@color/black"
	android:enabled="false"
	app:layout_constraintStart_toStartOf="parent"
	app:layout_constraintBottom_toBottomOf="parent"
	app:layout_constraintEnd_toStartOf="@+id/stopButton" />

<Button
	android:id="@+id/stopButton"
	android:layout_width="90dp"
	android:layout_height="60dp"
	android:layout_marginStart="40dp"
	android:layout_marginBottom="60dp"
	android:backgroundTint="@android:color/holo_red_light"
	android:text="@string/stop"
	android:enabled="false"
	android:textColor="@color/black"
	app:layout_constraintBottom_toBottomOf="parent"
	app:layout_constraintStart_toEndOf="@+id/startButton" />
```


The reason why I opted to keep a reload button in there is because I wanted to separate the launching Arti and checking the connection status in the code. This was mainly inspired by the fact that there was already a method in the code that just checked the connection status, so it made more sense to make a separate one that would do the actual launching. So, the start button would call the startArti() and then checkConnectionStatus(), but the reload button would only call checkConnectionStatus().

```java

 private void startArti() {
      textView.setText(R.string.intro_text);
      switch (selectedOption){

          case NO_PT:
              ((App)getApplication()).connectTorDirect();
              break;

          case OBFS4:
              List<String> lyreBirdBridgeLines = collectInputs();
              if (lyreBirdBridgeLines.isEmpty()) break;

              ((App) getApplication()).connectWithLyrebird(Integer.parseInt(obfs4Port.getText().toString()), lyreBirdBridgeLines);

              break;

          case SNOWFLAKE:
              String stunServers = stunServerInput.toString();
              String target = targetInput.toString();
              String front = frontInput.toString();
              List<String> snowflakeBridgesLines = collectInputs();

              if (snowflakeBridgesLines.isEmpty()) break;

              ((App) getApplication()).connectWithSnowflake(stunServers, target, front, snowflakeBridgesLines);

          default:
              break;
      }

      Handler handler = new Handler();
      handler.postDelayed(new Runnable() {
          public void run() {
              checkConnection();
          }
      }, 2000);
}

private void checkConnection() {

  fab.setEnabled(false);
  fabSpin.setDuration(1000*60).rotationBy((float) (1000 * 60) /4).setInterpolator(new LinearInterpolator()).start();

  connectionStatus.setVisibility(View.VISIBLE);
  connectionStatus.setText(R.string.performing_request);

  new CheckTorConnectionTask(this).execute();
}
```

I also implemented a ScrollView to display the log output. Since I had already learned how to make things in the App.java and MainActivity.java interact, this was fairly easy. I also really enjoyed this part because I got to figure out how to make it clearly look like code output. I ended up creating some drawable resources for this, and was quite pleased with the end result. I also made it scroll down automatically as new log output appeared, which was very satisfying. As a finishing touch, I also added a toggle to show and hide it, because I quickly realized that it might get annoying for the user. The difficult bit for the log output was figuring out how to constrain the box depending on which PT was selected, since they all had different numbers of input fields. This was a pain when I first implemented it, but then ended up becoming much simpler when I wrapped those input fields in another view (more on that later).


If the user does everything correctly up until this point, when they press the start button, there will first appear a “processing request” message right below the buttons at the bottom. Then, in accordance with whether and how the user was able to connect to Arti, a message is displayed.


When I went to implement the stop button, I realized that functionality had actually not yet been implemented in the ArtiProxy library! I brought this up with the team, and created an [issue](https://gitlab.com/guardianproject/tormobile/arti-mobile-ex/-/issues/6) to get it implemented. My colleague who was working on it kindly allowed me to watch and occasionally give my input while he dug through the Arti documentation to try and figure out how to do it. Pair programming is something that I am quite familiar with from college courses, having worked with a partner for many a lengthy assignment. It is something that I always enjoyed, and doing it here was no different. I have found that working with someone can be great for really tedious tasks because you can bounce ideas off each other and get through it much more easily. It’s a bit more difficult when it’s remote, but nonetheless a more enjoyable experience than slogging through a large codebase on one’s own.


While I was waiting for the stop functionality, I also got to work on giving the user a way to add and remove bridge lines in the Arti app. I learned that an average user might want to use even dozens of bridges at a time, so I had to think about what a good way to do it might be UI-wise. I settled on putting my input fields in a ScrollView, along with buttons to add and remove bridge lines. I set a max height for this ScrollView, making it automatically scroll down when the user exceeds that height (just like for the log output). I found that just wrapping the input fields in the ScrollView resulted in an error, since a ScrollView can only have one direct child. As a result, I had to wrap the input fields in a LinearLayout first, which meant I had to do the (frankly bothersome) work of redoing the constraints they had on them. As for the buttons, I thought it would look best to have them at the bottom, by the bridge line input field, each taking up half of the parent view horizontally. In order to place them next to each other horizontally, I had to wrap them in another horizontally-oriented LinearLayout. I also used a property called [weightSum](https://developer.android.com/reference/android/widget/LinearLayout#attr\_android:weightSum) to ensure that they would each take up half of the space. While having nested layouts is generally not recommended, I found that for this case it made sense. The nesting was not too deep, and the MainActivity.java code was a lot tidier with the input fields wrapped in that layout. The directly-one-after-another nature of the input field lended itself nicely to the LinearLayout, so it was all for the best.


The nested XML code:

```xml
<ScrollView
  android:id="@+id/inputScrollView"
  android:layout_width="match_parent"
  android:visibility="gone"
  android:layout_height="0dp"
  android:layout_marginStart="5dp"
  android:layout_marginEnd="10dp"
  android:layout_marginTop="5dp"
  app:layout_constraintHeight_max="220dp"
  app:layout_constraintTop_toBottomOf="@id/spinner"
  app:layout_constraintEnd_toEndOf="parent"
  app:layout_constraintStart_toStartOf="parent">

 <LinearLayout
   android:id="@+id/inputFieldsContainer"
   android:layout_width="match_parent"
   android:layout_height="0dp"
   android:orientation="vertical"
   tools:layout_editor_absoluteX="0dp">
<EditText
       android:id="@+id/obfs4Port"
       android:layout_width="match_parent"
       android:layout_height="wrap_content"
       android:layout_marginStart="15dp"
       android:layout_marginEnd="15dp"
       android:backgroundTint="@color/teal_200"
       android:ems="10"
       android:hint="@string/enter_obfs4_port"
       android:inputType="text"
       android:visibility="gone"
       app:layout_constraintEnd_toEndOf="parent"
       app:layout_constraintStart_toStartOf="parent" />
…

<LinearLayout
   android:layout_height="0dp"
   android:layout_width="match_parent"
   android:layout_weight="1"
   android:weightSum="2"
   android:orientation="horizontal">

<Button
       android:id="@+id/buttonAdd"
       android:layout_width="0dp"
       android:layout_height="match_parent"
       android:layout_weight="1"
       android:layout_marginEnd="15dp"
       android:layout_marginStart="15dp"
       app:layout_constraintEnd_toEndOf="parent"
       app:layout_constraintStart_toStartOf="parent"
       android:text="@string/add_bridge_line"/>

…

</LinearLayout>
</LinearLayout>
</ScrollView>
```



At this point, I was beginning to become annoyed with how messy one of my switch statements was becoming–the one that was controlling which views should be visible and which should be “gone” when a particular PT was selected. So, to clean it up, I decided to create a few private methods that would set those visibilities, then I just called them from the switch statement.


```java
private void onSelectionChanged(SelectedPluggableTransport s) {

  ConstraintSet constraintSet = new ConstraintSet();

  switch (s) {

      case NO_SELECTION:
          setDefaultVisibilities();
          constraintSet.clone(constraintLayout);
          break;

      case NO_PT:
          setDirectVisibilities();
          constraintSet.clone(constraintLayout);
          constraintSet.connect(logLabel.getId(), ConstraintSet.TOP, spinner.getId(), ConstraintSet.BOTTOM);
          break;

      case OBFS4:
          setLyrebirdVisibilities();
          constraintSet.clone(constraintLayout);
          constraintSet.connect(logLabel.getId(), ConstraintSet.TOP, inputScrollView.getId(), ConstraintSet.BOTTOM);
          break;

      case SNOWFLAKE:
          setSnowflakeVisibilities();
          constraintSet.clone(constraintLayout);
          constraintSet.connect(logLabel.getId(), ConstraintSet.TOP, inputScrollView.getId(), ConstraintSet.BOTTOM);
          break;
  }
  
  constraintSet.applyTo(constraintLayout);
}


private void setDefaultVisibilities() {
	noSelection.setVisibility(View.VISIBLE);
	logScrollView.setVisibility(View.GONE);
	logLabel.setVisibility(View.GONE);
	inputScrollView.setVisibility(View.GONE);
	startButton.setEnabled(false);
	fab.setEnabled(false);
}


private void setDirectVisibilities() {
	stunServerInput.setVisibility(View.GONE);
	obfs4Port.setVisibility(View.GONE);
	targetInput.setVisibility(View.GONE);
	frontInput.setVisibility(View.GONE);
	bridgeLineInput.setVisibility(View.GONE);
	noSelection.setVisibility(View.GONE);
	logLabel.setVisibility(View.VISIBLE);
	inputScrollView.setVisibility(View.GONE);
	addBridgeLine.setVisibility(View.GONE);
	startButton.setEnabled(true);
	fab.setEnabled(true);
}


private void setLyrebirdVisibilities() {
	bridgeLineInput.setVisibility(View.VISIBLE);
	obfs4Port.setVisibility(View.VISIBLE);
	stunServerInput.setVisibility(View.GONE);
	targetInput.setVisibility(View.GONE);
	frontInput.setVisibility(View.GONE);
	noSelection.setVisibility(View.GONE);
	logLabel.setVisibility(View.VISIBLE);
	inputScrollView.setVisibility(View.VISIBLE);
	addBridgeLine.setVisibility(View.VISIBLE);
	startButton.setEnabled(true);
	fab.setEnabled(true);
}

private void setSnowflakeVisibilities() {
	bridgeLineInput.setVisibility(View.VISIBLE);
	stunServerInput.setVisibility(View.VISIBLE);
	targetInput.setVisibility(View.VISIBLE);
	frontInput.setVisibility(View.VISIBLE);
	noSelection.setVisibility(View.GONE);
	obfs4Port.setVisibility(View.GONE);
	logLabel.setVisibility(View.VISIBLE);
	inputScrollView.setVisibility(View.VISIBLE);
	addBridgeLine.setVisibility(View.VISIBLE);
	startButton.setEnabled(true);
	fab.setEnabled(true);
}
```


Also to make my code more readable in general, going off of something the person who worked on the project before me had done, I created an enum called SelectedPluggableTransport to use in my switch statements rather than just the numbers. Thinking through and implementing all these readability and style improvements was one of my favorite parts of the process. Since I knew I was developing for another developer, getting this right felt important.


In the code, whenever a user added a new input field, it would be added to a List\<EditText> array. Later on, when they went to press the start button, startArti() would call a method named collectInputs(), which would iterate through the EditText array and return an array of their inputs. That would then be passed to the relevant App.java function as the bridge line array.


Overall, the process was quite fun! While I often got stuck and had to re-implement a bunch of stuff a few times, it was really satisfying when I figured out something that worked. I definitely feel more comfortable with Android Studio now, and could implement a basic app with much less trouble now.
