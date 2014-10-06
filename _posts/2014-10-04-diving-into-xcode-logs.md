---
layout: post
title: Diving Into Xcode Logs
---

At the Flatiron School we have been building a tool to collect the results from unit/integration tests that are included with our labs. The end goal of all of this was to have a `POST` request sent with the JSON formatted results of the tests. Having this data is mostly just to see where students are right now, but down the road will lead to some great stuff. I'm excited. But! to get access to the students test results in Objective-C was not simple. Here is the saga of how we do it.

## How It Works

Each lab has a Test Post-Action and a test runner shell script. The Test Post Action gets the Derived Data Path (`$BUILD_DIR`) and goes into the Test Logs folder. Then the post-action passes that log Derived Data Directory and the location of source of the project (`$SRCROOT`) to a shell script. Then the test runner shell script ungunzips the `.xcactivitylog`, extracts out just the testing output and send it to `xcpretty`. We then have a reporter written for xcpretty that formats everything into a JSON hash and send a `POST` request to our server.

## How We Got There

### XCPretty

There have been two tools to allow command line running of tests with some sort of formatting attached to it. The first that came out of facebook was [xctool](https://github.com/facebook/xctool/). It was pretty great but it was a complete replacement for `xcodebuild`. I didn't really want that. I wanted a tool that could just take the output of `xcodebuild` and bring it into ruby so that I could format it. Thankfully! [xcpretty](https://github.com/supermarin/xcpretty) exists. It's specifically made to just parse out the results of `xcodebuild`. Even better...it has a reporters functionality that allows you to format the results in different formats. Taking a look at the formatter for the [junit](https://github.com/supermarin/xcpretty/blob/master/lib/xcpretty/reporters/junit.rb) I noticed methods like this

```ruby
def format_passing_test(classname, test_case, time)
      test_node = suite(classname).add_element('testcase')
      test_node.attributes['classname'] = classname
      test_node.attributes['name']      = test_case
      test_node.attributes['time']      = time
      @test_count += 1
end
```

That looked perfect. There is a huge [list of events](https://github.com/supermarin/xcpretty/blob/master/lib/xcpretty/parser.rb#L210-278) that the xcpretty parser will listen to. This allowed me to get access to any event going on in an `xcodebuild` without having to write all of the [annoying regex](https://github.com/supermarin/xcpretty/blob/master/lib/xcpretty/parser.rb#L5-L187) that I was convinced I was going to have to write, it was seriously perfect. So I wrote a custom reporter that borrowed heavily from the JUnit parser. All it cared about was: start of tests, passing test, failing test, pending test and on finishing tests it serialized the hash into JSON and submitted a POST request to our server. We also included a bit of meta-data on the repository that was currently being run. This included things like github name and github user id (stored in a `.netrc` file) and using the [Git](https://github.com/schacon/ruby-git) gem to understand which lab they are working on. 

### From Command Line to XCode

So now I was able to get test results if students ran the tests in the command line with a very annoying command like this:

```
xcodebuild -workspace yourworkspace.xcworkspace/ -scheme yourscheme test -sdk iphonesimulator8.0 | xcpretty -t --report flatiron
```

This was neither ideal nor really how iOS developers work. Command line test running is really a feature to be used for Continuous Integration, not day-to-day tests. Also I wanted as close as possible to perfect data collection. In iOS development, that meant retrieving the test results from when we type `CMD-u`. My first thought was to just have the tests get re-run using a test post-action...but that ended poorly because it made tests take forever and the simulator had to come up twice. Thankfully I remembered the log viewer window in XCode. If you click on the `Logs` tab and hit the hamburger like icon on the far right of each line you get a text output. Hark! A log file. Even better it looks identical to the `xcodebuild` output which `xcpretty` could parse. This text must exist somewhere. A quick spotlight search didn't turn anything up. I then did a search using `find` and couldn't find anything. Finally I just went to my Derived Data folder to see what's there.

### Derived Data

The Derived Data folder is where all of the by-products of compilation goes. It's located in `~/Library/Developer/Xcode/DerivedData` and contains a TON of folders. Once you go into one of you app-specific folders there is a sub folder called `Logs/Test` which contains a bunch of (seemingly randomly named) `.xcactivitylog` files. Opened them up in vim and noticed it was a binary file. Thanks Apple. Thanks to [this](http://stackoverflow.com/questions/13861658/is-it-possible-to-search-though-all-xcodes-logs) StackOverflow article I discovered these were just `gz` archives. Un-gunzipped them and **there was the log file**. Annoyingly it used windows style returns for some reason but that's ok. Next up was figuring out this log file.

### The xcactivitylog

This log file had *a ton* of junk in it. Things like this:

```
SLF07#21%IDEActivityLogSection1@0#36"com.apple.dt.unit.cocoaUnitTestSmall25"Test ObjectOrientedPeople25"Test ObjectOrientedPeople4f917b7d28dbb941^8387098128dbb941^1(29%IDEActivityLogUnitTestSection2@1#31"com.apple.dt.unit.cocoaUnitTest37"Test target ObjectOrientedPeopleTests37"Test target ObjectOrientedPeopleTests0e166a7e28dbb941^8882098128dbb941^1(2@3#35"com.apple.dt.IDE.UnitTestLogSection24"Run test suite All tests24"Run test suite All tests817c2d8028dbb941^4b39b38028dbb941^1(2@3#35"com.apple.dt.IDE.UnitTestLogSection47"Run test suite ObjectOrientedPeopleTests.xctest47"Run test suite ObjectOrientedPeopleTests.xctest25cb2d8028dbb941^ac1fb38028dbb941^1(2@3#35"com.apple.dt.IDE.UnitTestLogSection25"Run test suite PersonSpec25"Run test suite PersonSpecf1d62d8028dbb941^c635a68028dbb941^18(2@3#35"com.apple.dt.IDE.UnitTestLogSection124"Run test case test_Person__grow__should_grow_a_random_amount_between_0_and_1_inch_if_person_is_a_girl_less_than_11_years_old124"Run test case test_Person__grow__should_grow_a_random_amount_between_0_and_1_inch_if_person_is_a_girl_less_than_11_years_olde3e12d8028dbb941^3603948028dbb941^-518"Test Suite 'PersonSpec' started at 2014-09-30 18:05:52 +0000
```

I can see there is good stuff in there but it's nothing that `xcpretty` can take in. After a little bit of comparing the log file to what I saw in XCode I found that at the bottom there was what I needed. Problem was it was logged out over and over again. I'm not sure why it was duplicated but I used this regular expression to get only what I needed `/(^Test Suite '[\w-]+?\.xctest' started at .+?Test Suite '[\w-]+?\.xctest' (failed|passed).+?\.$)/m`. Here it is!

```
Test Suite 'ObjectOrientedPeopleTests.xctest' started at 2014-09-30 18:05:52 +0000
Test Suite 'PersonSpec' started at 2014-09-30 18:05:52 +0000
Test Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_0_and_1_inch_if_person_is_a_girl_less_than_11_years_old]' started.
2014-09-30 14:05:52.432 ObjectOrientedPeople[13961:325276] CLTilesManagerClient: initialize, sSharedTilesManagerClient
2014-09-30 14:05:52.432 ObjectOrientedPeople[13961:325276] CLTilesManagerClient: init
2014-09-30 14:05:52.432 ObjectOrientedPeople[13961:325276] CLTilesManagerClient: reconnecting, 0x79e874b0
Test Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_0_and_1_inch_if_person_is_a_girl_less_than_11_years_old]' passed (0.280 seconds).
Test Case '-[PersonSpec test_Person__addFriends__should_add_Person_objects_to_friends_array]' started.
Test Case '-[PersonSpec test_Person__addFriends__should_add_Person_objects_to_friends_array]' passed (0.000 seconds).
Test Case '-[PersonSpec test_Person__grow__should_grow_0_inches_if_person_is_a_boy_older_than_16]' started.
Test Case '-[PersonSpec test_Person__grow__should_grow_0_inches_if_person_is_a_boy_older_than_16]' passed (0.000 seconds).
Test Case '-[PersonSpec test_Person__removeFriends__should_return_an_array_of_removed_friends]' started.
Test Case '-[PersonSpec test_Person__removeFriends__should_return_an_array_of_removed_friends]' passed (0.001 seconds).
Test Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_0_and_1_inch_if_person_is_a_boy_less_than_12_years_old]' started.
Test Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_0_and_1_inch_if_person_is_a_boy_less_than_12_years_old]' passed (0.011 seconds).
Test Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_5_and_2_inches_if_person_is_a_girl_11_or_older_and_15_or_younger]' started.
Test Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_5_and_2_inches_if_person_is_a_girl_11_or_older_and_15_or_younger]' passed (0.000 seconds).
Test Case '-[PersonSpec test_Person__generatePartyList__should_return_a_string]' started.
Test Case '-[PersonSpec test_Person__generatePartyList__should_return_a_string]' passed (0.000 seconds).
Test Case '-[PersonSpec test_Person__generatePartyList__should_print_a_list_of_friends_seperated_by_commas]' started.
Test Case '-[PersonSpec test_Person__generatePartyList__should_print_a_list_of_friends_seperated_by_commas]' passed (0.000 seconds).
Test Case '-[PersonSpec test_Person__grow__should_grow_0_inches_if_person_is_a_girl_older_than_15]' started.
Test Case '-[PersonSpec test_Person__grow__should_grow_0_inches_if_person_is_a_girl_older_than_15]' passed (0.001 seconds).
Test Case '-[PersonSpec test_Person__removeFriend__Should_remove_a_found_friend_from_the_friends_array]' started.
Test Case '-[PersonSpec test_Person__removeFriend__Should_remove_a_found_friend_from_the_friends_array]' passed (0.000 seconds).
Test Case '-[PersonSpec test_Person__removeFriends__should_remove_found_friends_from_friends_array]' started.
Test Case '-[PersonSpec test_Person__removeFriends__should_remove_found_friends_from_friends_array]' passed (0.000 seconds).

Test Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_5_and_35_inches_if_person_is_a_boy_12_years_or_older_and_16_years_or_younger]' started.
Testt Case '-[PersonSpec test_Person__grow__should_grow_a_random_amount_between_5_and_35_inches_if_person_is_a_boy_12_years_or_older_and_16_years_or_younger]' passed (0.024 seconds).
Test Case '-[PersonSpec test_Person__initializers__should_initialize_with_name_passed_in_and_height_at_9]' started.
Test Case '-[PersonSpec test_Person__initializers__should_initialize_with_name_passed_in_and_height_at_9]' passed (0.001 seconds).
Tesct Case '-[PersonSpec test_Person__removeFriend__Should_return_NO_if_the_friend_was_not_found_and_not_removed]' started.
Test Case '-[PersonSpec test_Person__removeFriend__Should_return_NO_if_the_friend_was_not_found_and_not_removed]' passed (0.001 seconds).
Test Case '-[PersonSpec test_Person__initializers__should_initalize_height_to_9_and_name_should_be_an_empty_string]' started.
Test Case '-[PersonSpec test_Person__initializers__should_initalize_height_to_9_and_name_should_be_an_empty_string]' passed (0.001 seconds).
Test Case '-[PersonSpec test_Person__addFriends__should_add_an_array_of_Person_friends_to_a_Persons_friends_array]' started.
Test Case '-[PersonSpec test_Person__addFriends__should_add_an_array_of_Person_friends_to_a_Persons_friends_array]' passed (0.000 seconds).
Test Case '-[PersonSpec test_Person__removeFriend__Should_return_YES_if_the_friend_was_found_and_removed]' started.
Teast Case '-[PersonSpec test_Person__removeFriend__Should_return_YES_if_the_friend_was_found_and_removed]' passed (0.001 seconds).
Test Case '-[PersonSpec test_Person__initializers__should_have_an_initalizer_called_initWithName]' started.
T est Case '-[PersonSpec test_Person__initializers__should_have_an_initalizer_called_initWithName]' passed (0.000 seconds).

Test Suite 'PersonSpec' passed at 2014-09-30 18:05:52 +0000.
	 Executed 18 tests, with 0 failures (0 unexpected) in 0.323 (0.372) seconds

Test Suite 'ObjectOrientedPeopleTests.xctest' passed at 2014-09-30 18:05:52 +0000.
```

### Parsing the logs after test running

I now had the log file, I just needed to in an automated way send it to xcpretty. Annoyingly there are a lot of steps. I wrote a script that we add to each assignment that does roughly this:

  1) cd to the derived data directory (passed in from test post-action)
  2) un-gunzip the latest `.xcactivitylog` file. I couldn't figure out the naming convention so I just did it by date
  3) convert windows returns into unix returns
  4) regex out the appropriate lines
  5) Send those lines to `xcpretty`

Major top tip when working with Post-Actions. The `stdout` and `stderr` get sent to `/dev/null` by default so we can't see if anything goes wrong. This is annoying. Thankfully we can use exec to get the log output. When I am debugging the post-action I put this line as the first line in the post-action run `exec > /tmp/my_log_file.txt 2>&1`. This spits all output to `/tmp/my_log_file.txt`.

### Room for Improvement

It's not perfect at all...but it works. Even more surprising it works pretty darn consistently. The biggest problem right now is understanding *which ruby* XCode decides to use to run the post-actions. On some systems its rvm, others it is the system ruby. I need to figure that out so I can reliably have students install our modified `xcpretty` gem onto the right version of gems.
