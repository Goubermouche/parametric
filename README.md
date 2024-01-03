## Parametric
Parametric is a basic header-only command-line parameter parser aiming to serve as a layer between the command line and your application. 

## Basic usage
After copying the contents of `parametric.h` you can simply including like so: 
```cpp
#include <parametric.h>
```
Parametric supports one main "parameter structure", which is basically a tree-like structure where the leaf nodes are individual "commands". Additionally, multiple commands can be stored in a "command group". 

### Commands
Commands are the basic building block for parametric, each command can have a number of flags and/or positional arguments. Each command is also declared using a given function - this function will be executed, if the user decides to run the relevant command. 

## Positional arguments
A given command may always need some *data*, we'd like this data to be structured in a certain way to make the usage of our program clearer to the user, this can be achieved using **positional arguments**. A positional argument is an argument of a given type, which the user **has** to specify, when running a given command (in other words, its a mandatory piece of data). The argument is additionally *positional*, which means that, when running the program, the argument is expected to be at the same position across all app runs. 

## Flags
Flags are additional pieces of data which can be supplied to a command. To supply a flag, the user needs to either specify a long flag alias, or, optionally, a short one (ie. `--long_alias` or `-l`). By default, flags only represent boolean values, in this case, each flags' existence implicitly represents that it is `true`, on the other hand, if the flag isn't specified (it doesn't exist) the value is implicitly set to `false`. Flags may also hold any other data, in this case the flag also expects a valid string immediately after it (ie. `--target arch`). Flags may be placed anywhere after the command name and the last positional argument, and their order does not matter

### Command groups
A command group is a more complex building block, which provides a way of adding additional layers to your program. Each command group can contain other command groups and/or regular commands. 

## Basic example 
Say we want to create an application for handling **rats**. This application can be separated into two main parts - **trading rats** and **viewing our rats**, in order to do this, we'll initialize our `program` and we'll get started on creating our **trading** command group: 
```cpp
#include <parametric.h>

int main(int argc, char** argv) {
    parametric::program program;

    // rat trading
    parametric::command_group& trading = program.add_command_group("trading", "Rat trading");

    // parse and execute the program
    return program.parse(argc, argv);
}
```
Since our command group is useless as of now, let's add some commands to it: 
```cpp
#include <parametric.h>

int list_rats(const parametric::parameters& params) {
	/* logic for listing all available rats */
	return 0;
}

int sell_rat(const parametric::parameters& params) {
	auto name = params.get<std::string>("name"); // parsed rat name
	auto price = params.get<double>("price"); // parsed rat price
	/* logic for selling a rat */
	return 0;
}

int main(int argc, char** argv) {
    parametric::program program;

    // rat trading
    parametric::command_group& trading = program.add_command_group("trading", "Rat trading");

	// listing existing rats
	trading.add_command("list", "List all available rats", list_rats);

	// selling rats
	parametric::command& sell = trading.add_command("sell", "Sell a rat", sell_rat);
	sell.add_positional_argument<std::string>("name", "Name of the rat to sell");
	sell.add_positional_argument<double>("price", "Price at which to sell the rat");

    // parse and execute the program
    return program.parse(argc, argv);
}
```
We've now declared a command group called "trading", which contains the "list" and "sell" commands. Each command has a name and a description. Additionally, each command has a function related to it (`list_rats` and `sell_rat`). As an example, when the user runs the following command: 
```shell
$ rats.exe trading list
```
the `list_rats` function will be executed. Each function also contains a `parameters` parameter. This parameter contains a map of all variables which have been specified by the user or which have been implicitly defined. Say the user runs the following command: 
```shell
$ rats.exe trading sell Ryan 12400
```
In this case, the command had two positional arguments: "name" and "price" - both of these arguments, and their parsed value, can be referenced using the `parameters` class. Let's move onto the "view" command: 
```cpp
enum class rat_location {
	FOREST, // view your rat in a forest 
	BEACH,  // view your rat on a beach
	LHC     // view your rat in the Large Hadron Collider
};

int view_rat(const parametric::parameters& params) {
	auto name = params.get<std::string>("name"); // parsed rat name

	if(params.contains("location")) {
		// the user would like to see their rat in a specific location
		auto location = params.get<rat_location>("location"); // parsed rat location
	}

	auto is_smirking = params.get("smirk");
    auto label = params.get<std::string>("label");

	/* logic for viewing a rat */
	return 0;
}

template<>
struct parametric::options_parser<rat_location> {
	static auto parse(const std::string& value) -> rat_location {
		if(value == "forest") {
			return rat_location::FOREST;
		}

		if (value == "forest") {
			return rat_location::FOREST;
		}

		if (value == "lhc") {
			return rat_location::LHC;
		}

		throw std::invalid_argument("invalid rat location");
	}
};
...

...
// rat viewing
parametric::command& view = program.add_command("view", "View one of your rats", view_rat);
view.add_positional_argument<std::string>("name", "Name of the rat to view");
view.add_flag<rat_location>("location", "Location to view the rat in", "l");
view.add_flag("smirk", "Specify whether the rat should be smirking");
view.add_flag<std::string>("label", "Specify the view label", "Rat");
...

```
We've created our view command, since we'd obviously like to view our rat in different locations, we'll utilize an enum (`rat_location`), since this enum and its conversion is unfortunately not defined in `parametric`, we'll have to define it ourself - for more info see the `options_parser`.
Notice that the location is a flag - meaning that, since it uses a non-boolean type, it does not have a default value - because of this we have to check whether the key is defined first, when handling the flag in `view_rat`.       
Additionally, we've also declared a "label" flag, this flag, however, has a default value (specified by the last parameter "Our cute rat"), because of this, we can access the "label" flag without checking for its existence first. Execution of the view command could now look something like this: 

```shell
# view our rat at the large hadron collider
$ rats.exe view Jonathan -l lhc --label Visit --smirk

# Notice that: 
# - the label can only be used with the long alias, since a shorter one was never declared
# - the smirk flag is used with no additional value, since its a boolean flag (a pure flag)
# - the location flag uses the short alias (-l)
```
## Help menu
Whenever the user's input suffers from incorrect formatting/missing positional arguments/missing commands/etc. Parametric automatically handles this behavior by displaying an error message and a help menu, after this the program exits with the exit code of '0'. The help menu contains a label block with potential candidates for commands/arguments/flags, the width of this menu can be controlled via the `PARAMETRIC_TABLE_WIDTH` macro.