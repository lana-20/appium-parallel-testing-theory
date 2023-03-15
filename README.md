# Running Appium Tests in Parallel in Theory


It's important to be able to run multiple tests at once. Eventually your testsuite will grow large enough that waiting for them to complete one by one just doesn't make sense. But running tests in parallel comes with a number of complications. Let's talk about everything that you'll need in order to successfully start running your testsuite in parallel.

Requirements for Parallel Testing
Test runner that can run multiple tests at the same time. Pytest can do this through the use of a
plugin called pytest-xdist
Multiple test browsers or devices (at least one per parallel execution thread). Can be tricky to scale
easily especially with mobile; cloud services help with this problem.
(Maybe) multiple webdriver servers. E.g., one Geckodriver instance per parallel execution thread.
Partitioning of system resources required for a test. E.g., each Android session requires the exclusive
use of a system port, so simultaneous sessions need to be configured not to use the same port.
Ensure total independence of tests (tests cannot assume state from any other tests)

1. The first thing you need to run your tests in parallel is a test runner that can run multiple tests at the same time. If your test runner can't run multiple tests at the same time, you're kind of dead in the water. And unfortunately, there are some common test runners out there that don't have this ability, so if you end up using one of those, you'll have to do some pretty intense gymnastics to figure out how to break your testsuite up into different processes. But luckily for us, Pytest's test runner can parallelize tests within a suite no problem, through the use of a special plugin called pytest-xdist.
2. The second thing you need is a number of test devices, or browsers. If you want to run 5 tests at a time in your iOS test suite, you need 5 iOS devices or simulators that can be independently used. Mobile devices pose the biggest challenge here, primarily because it's more computationally and financially expensive to run a mobile device than a web browser. Also, it's quite easy to run a number of different instances of the same browser on your system, but interacting with a number of identically-configured iOS or Android devices involves knowing some unique information about each of them, like an ID. This creates complexity in our test code, as we'll see. This is the area where a device or browser cloud can make life a lot easier. With a device or browser cloud, you don't have to figure out how to provision and maintain each browser or device. Instead, you just use capabilities to request devices that make sense, and as long as your account has access to them, you're good!
3. There's a third thing you might need, which is multiple webdriver servers to handle your multiple parallel sessions. Some webdriver servers, like Geckodriver, for example, are only able to handle one session at a time. Thus if you want to run multiple Firefox tests at the same time on a single computer, you'll need to first be running multiple Geckodriver servers on different ports, and figure out how to make sure each parallel test is only talking to one Geckodriver server. Other servers, like Appium or Selenium if you choose to run Selenium in front of your browser drivers, can handle multiple requests at the same time, so you don't need to worry about this.
4. Fourth, you need to make sure that each of your parallel sessions aren't competing for resources. What do I mean by resources? Well, one kind of resource could be a device. If two simultaneous sessions are trying to use the same device, you're going to get some very odd behavior and your tests will not work. But there are other kinds of resources that are more invisible to us as well. For example, Appium uses various means of communication in order to work with its drivers. For both the XCUITest and the UiAutomator2 drivers, Appium connects over a local network to talk to a special server running on the device. To facilitate this network connection, Appium uses a local port. A port is just a channel for communication that can be used by one process at a time. Normally when you're running just one test a time, you don't have to think about ports. Appium will just use the default port, and you'll be fine. But if you're running two tests at once, and Appium tries to talk to two devices over the same port, very weird things could happen. Luckily you can tell Appium which port to use via a special capability, so you can make sure that each of your sessions uses a different port. That way they don't step on each other's toes. Or shout in each other's ears, I guess would be a better analogy! We'll look at these capabilities momentarily.
5. Finally, you need to make sure that your tests are designed in such a way that they don't step on each other's toes either. What do I mean? Well, when we're initially designing a testsuite, and running tests one after the other, it's not too hard to get into a situation where some of the later tests depend on something that some of the earlier tests did. Imagine that you have a test that performs a login, and a test that runs after the login test that assumes the user is already logged in. This assumption might work when you run all your tests one after the other, but if the tests are all run in parallel on different devices, this assumption would cause the latter test to fail! What we want to strive for is tests that are totally independent from one another. What this means is that each test needs to set up its own state, which is a practice we'll talk about more later on.
