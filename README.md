burlap_rosbridge
================

A BURLAP library extension for interacting with robots run on ROS by creating BURLAP `Environment` instances that maintain state and execution actions by ROS topics communicated via ROS Bridge.

##Linking
burlap_rosbridge is indexed on Maven Central, so if you want to merely use it, all you need to do is include in the `<dependencies>` section of your project's pom.xml file:
(As of writing this, 2.2.0 is not yet on Maven Central as it is not finished)
```
<dependency>
  <groupId>edu.brown.cs.burlap</groupId>
  <artifactId>burlap_rosbridge</artifactId>
  <version>2.2.0</version>
</dependency>
```
and it will automatically be downloaded. Note that you will also want to explicitly include BURLAP (also on Maven Central) because BURLAP is set to not link transitively through burlap_rosbridge. This choice was made to make it clear which version of BURLAP you wanted to use in your project. To link to BURLAP, add
```
<dependency>
  <groupId>edu.brown.cs.burlap</groupId>
  <artifactId>burlap</artifactId>
  <version>2.2.0-SNAPSHOT</version>
</dependency>
```
or switch its version to whatever is appropriate.

Alternatively, you may compile and install the code directly (or modify as needed), as described in the compiling section of this readme.


##Description of Code
Currently, there are two abstract `Environment` implementations, `AbstractRosEnvironment` and a slightly more concrete implementation, `RosEnvironment`. `AbstractRosEnvironment` provides the infrastructure for managing action execution (via the `Environment` `executeAction(GroundedAction)` method). In short, `AbstractRosEnvironment` allows you to specify `ActionPublisher` objects for each BURLAP `Action` that are responsible for publishing the action events to ROS. There are a number of included implementations of `ActionPublisher` in the library already, but the framework is made to enable you to implement your own. The way an `ActionPublisher` is implemented also affects whether action execution is synchronous or asynchronous.

The `AbstractRosEnvironment` does not, however, maintain the state of the Environment, which is a task left for the concrete implementations of it. The `RosEnvironment` extends `AbstractRosEnvrionment` with additional methods in which there is a single ROS topic that is publishing the state of the environment. Rewards and termination conditions for the environment are then handled by specifying standard BURLAP RewardFunction and TerminalFunction implementations that will operate on the state. Implementing `RosEnvironment` requires implementing a single method: `State unpackStateFromMsg(JsonNode data, String stringRep)` which is given the JSON message of the ROS topic and should turn it into a BURLAP `State` object that is returned. You may want to use the provided `MessageUnpacker` class if you have an implemented `State` object whose datastructure matches the ROS message and can be straightforwardly unpacked.


If you would prefer to have BURLAP code create the `State` from various ROS topics (rather than having a single ROS topic that is publishing it), then you may want to create your own extension of `AbstractRosEnvironment`.

Although BURLAP is currently compatible with Java 6, You will need Java 7 to use this library because the ROS Bridge Websocket connection (provided by our [java_rosbridge](https://github.com/h2r/java_rosbridge) library) uses Jetty 9, which requires Java 7.

See the Java doc and example code below for more informaiton.

##Compiling

Compiling and management is now performed with Maven. If you would like to compile with ant, use the ant branch. However, going forward all future updates will requrie Maven. If you do not have Maven, you can get it from https://maven.apache.org/download.cgi

To compile use

```
mvn compile
```
Create a target jar and Java doc with

```
mvn package
```

Install the jar into your local repoistory with

```
mvn install
```

Link to burlap_rosbridge from a project by adding the following to the `<dependencies>` section of your project's pom.xml file.

```
<dependency>
  <groupId>edu.brown.cs.burlap</groupId>
  <artifactId>burlap_rosbridge</artifactId>
  <version>2.2.0-SNAPSHOT</version>
</dependency>
```


##Example code
We provide two sets of example code. One is more straightforward for testing purposes. The latter shows you how to control a ROS robot that responds to Twist messages.

###Example 1
```
public class Example1 {

	public static class StringState implements State{

		public String data;

		public StringState() {
		}

		public StringState(String data) {
			this.data = data;
		}

		@Override
		public List<Object> variableKeys() {
			return Arrays.<Object>asList("data");
		}

		@Override
		public Object get(Object variableKey) {
			return data;
		}

		@Override
		public State copy() {
			return new StringState(data);
		}

		@Override
		public String toString() {
			return data;
		}
	}

	public static void main(String[] args) {

		SADomain domain = new SADomain();
		new NullAction("action1", domain);
		new NullAction("action2", domain);

		//setup ROS information
		String uri = "ws://localhost:9090";
		String stateTopic = "/burlap_state";
		String stateMessage = "std_msgs/String";
		String actionTopic = "/burlap_action";


		RosEnvironment env = new RosEnvironment(domain, uri, stateTopic, stateMessage) {
			@Override
			public State unpackStateFromMsg(JsonNode data, String stringRep) {
				MessageUnpacker<StringState> unpacker = new MessageUnpacker<StringState>(StringState.class);
				return unpacker.unpackRosMessage(data);
			}
		};
		env.setActionPublisherForMultipleAcitons(domain.getActions(), new ActionStringPublisher(actionTopic, env.getRosBridge(), 500));
		env.setPrintStateAsReceived(true);

		Policy randPolicy = new RandomPolicy(domain);
		randPolicy.evaluateBehavior(env, 100);

	}

}

```


In this example code we assume that ROS is being run on the local host (port 9090 as default for ROS Bridge)
and that there is a ROS topic named `/burlap_state` that is defined by a single string state variable. Generally, your state message from ROS would be something much more expressive than a string message, but we're using something as simple a string for demonstration purposes. For testing, you can have ROS publish a dummy burlap_state string message that simply has the string value "hello" with the following ros command:

`rostopic pub /burlap_state std_msgs/String -r 1 -- 'hello'`

The first thing we do in our example code is define a cooresponding BURLAP `State` definition for a state define by a single string value. Note that to make ROS message unpacking trivial, we define the State class ti use public data fields that perfectly match the ROS data structure for the std_msgs/String message; that is, it is defined by a single String field named `data`. Furthermore, we make sure that we have a default constructor for de-serialization.

In our main method, we begin by constructing an example domain that consists of two actions, "action1" and "action2". Next we define the string values for our ROS set up (e.g., ROSBridge address--which we assume to be the localhost--and message topics/types. Then we implement our subclass of RosEnvironment that turns the ROS string message into our `StringState` using the java_rosbridge `MessageUnpacker`. We also tell our RosBridge environment to use a String publisher for action executions, which means the action name will be set as a String message to the ROS topic, and set the duration of an action to be for 500ms (that is, the action execution in BURLAP will block for 500ms, giving ROS time to execute the action message).

Finally, we setup a `RandomPolicy` in BURLAP and have it run for 100 steps in our ROS environment.

When you run the code, you should find that it's printing the "hello" State string the environment is generating from the message it's receiving from ROS. And on the command line of the ROS computer, if you enter the command
```
rostopic echo /burlap_action
```
you should see it printing out the random action choices from our random policy.

###Example 2
In the last example we setup an environment that published actions as string representations of the action name. This approach is only effective if you have some ROS code running that knows how to interpret the string representations
and actuate it on the robot, thereby requiring a middle man. A more direct way to control the robot is to have
action execution publish more typical ROS messages that specify the actuation. For example, on turtlebot robots, and various other robots, it is common to publish a ROS Twist message over a period of time to have the robot move. In this example code, we setup the BURLAP environment so that action execution results in publishing a Twist message for a fixed duration, thereby directly controlling the robot without a middle-man ROS script. We also then use a `EnvironmentShell` over the Environment we created so that you can manually control the robot through the terminal to test it out. In this case, we won't bother parsing a `State` since we are just demonstrating more advanced action execution and instead will tell our Environment to simply always return a `NullState`.

```
public class Example2 {

	public static void main(String [] args){
	
		//create a new domain with no state representation
		Domain domain = new SADomain();
	
		//create action specification (we don't need to define transition dynamics since we won't be doing any planning)
		new NullAction("forward", domain); //forward
		new NullAction("backward", domain); //backward
		new NullAction("rotate", domain); //clockwise rotate
		new NullAction("rotate_ccw", domain); //counter-clockwise rotate
	
		//define the relevant twist messages that we'll use for our actions
		Twist fTwist = new Twist(new Vector3(0.1,0,0.), new Vector3()); //forward
		Twist bTwist = new Twist(new Vector3(-0.1,0,0.), new Vector3()); //backward
		Twist rTwist = new Twist(new Vector3(), new Vector3(0,0,-0.5)); //clockwise rotate
		Twist rccwTwist = new Twist(new Vector3(), new Vector3(0,0,0.5)); //counter-clockwise rotate
	
		//setup ROS information
		String uri = "ws://localhost:9090";
		String stateTopic = "/burlap_state";
		String stateMessage = "std_msgs/String"; //state topic and type irrelevant for this example
		String actionTopic = "/mobile_base/commands/velocity"; //set this to the appropriate topic for your robot!
		String actionMsg = "geometry_msgs/Twist";
	
	
		//create environment
		RosEnvironment env = new RosEnvironment(domain, uri, stateTopic, stateMessage) {
			@Override
			public State unpackStateFromMsg(JsonNode data, String stringRep) {
				return NullState.instance;
			}
		};
		
		//force the environment state to a NullState so we don't have to setup a burlap_state topic on ROS
		env.overrideFirstReceivedState(NullState.instance);
	
		int period = 500; //publish every 500 milliseconds...
		int nPublishes = 5; //...for 5 times for each action execution...
		boolean sync = true; //...and use synchronized action execution
		env.setActionPublisher("forward", new RepeatingActionPublisher(actionTopic, actionMsg, env.getRosBridge(), fTwist, period, nPublishes, sync));
		env.setActionPublisher("backward", new RepeatingActionPublisher(actionTopic, actionMsg, env.getRosBridge(), bTwist, period, nPublishes, sync));
		env.setActionPublisher("rotate", new RepeatingActionPublisher(actionTopic, actionMsg, env.getRosBridge(), rTwist, period, nPublishes, sync));
		env.setActionPublisher("rotate_ccw", new RepeatingActionPublisher(actionTopic, actionMsg, env.getRosBridge(), rccwTwist, period, nPublishes, sync));
	
		//create a shell to control the turtlebot with the ex command
		EnvironmentShell shell = new EnvironmentShell(domain, env);
		shell.start();
	}

}
```
The first thing we do is create a domain with actions for rotating in either direction, going forward, backwards, or doing nothing. Next we create Twist objects that are part of the [java_rosbridge](https://github.com/h2r/java_rosbridge) library we are using. These objects are Java Beans that follow the same structure as the ROS Twist message: a linear `Vector3` component and an angular `Vector3` component. Note that if you were writing code for a different kind of robot that didn't respond to Twist messages and used a message type not in [java_rosbridge](https://github.com/h2r/java_rosbridge), you could simply create your own [Java Bean](https://en.wikipedia.org/wiki/JavaBeans) class for the ROS message and use that just as well.

We then setup a `RosEnvironment` in a similar way as before, but this time for simplicity of demonstration, note that we just return a `NullState`. We also tell the `RosEnvironment` not to block on waiting for a state message to be received since it won't be receiving one and immediately set the state to a `NullState`.

We then set up a `ActionPublisher` for each of our BURLAP actions. Specifically, we use a `RepeatingActionPublisher`. A `RepeatingActionPublisher` will have the affect of publishing a specified message a fixed number of times at a specified period. Specifically, for each BURLAP action, we define a `RepeatingActionPublisher` that will publish the corresponding Twist message 5 times with a period of 500 milliseconds between each publish. We also have to specify the action topic for this message. We set it to the topic commonly used by the Turtlebot robot, but you should be sure set it to whichever topic your Twist-controlled robot will respond. Note that we set a synchronized flag for the  `RepeatingActionPublisher` publisher to true. This has the effect of the `ActionPublisher` blocking until it has published all 5 messages before returning control to the calling `Environment`. Because we set it to be synchronized, it will also automatically set the return delay to the period length, which will cause the `RosEnvironment` to wait an additional 500 millseconds after the `RepeatingActionPublisher` published its final 5th message thereby allowing time for that final message to have an affect.

Finally, we use the standard BURLAP `EnvironmentShell` to allow us to manually interact with the `Environment` with keyboard commands. When you run the code, the shell should start and you should be able to use the `ex` command to specify acitons to execute. For example, try `ex forward`.


#### Large Message Sizes

Some state messages from ROS might be very large, such as frames from a video feed. In these cases, it's likely that the message size is larger than what Jetty's websocket buffer size is by default. However, you can increase the buffer size by subclassing `RosBridge` and annotating the subclass to have a larger buffer size. You do not need to implement or override any methods; `RosBridge` is subclassed purely to give it a custom buffer size annotation. For example:

```
@WebSocket(maxTextMessageSize = 500 * 1024) public class BigMessageRosBridge extends RosBridge{}
```

If you then instantiate your subclass, connect with it, and use a AbstractRosEnvironment or RosEnvironment constructor that accepts a RosBridge instance (rather than one that asks for the URI), then your environment will be able to handle the larger message sizes. See the readme and Java doc for [java_rosbridge](https://github.com/h2r/java_rosbridge) for more information.


### RosShellCommand

burlap_rosbridge also comes with an additional `ShellCommand` for interacting with RosBridge that you can add to your shells called `RosShellCommand`. This command allows you to echo a topic on ROS, publish to a topic, or a send a raw message to the RosBridge server. After adding it to your shell, see it's help with the -h option for more information. For example, add it to your shell as follows:
```
EnvironmentShell shell = new EnvironmentShell(domain, env);
shell.addCommand(new RosShellCommand(env.getRosBridge()));
shell.start();
```
