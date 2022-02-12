# Cloud Shell and gcloud

## Environment Configuration


To see what your default region and zone settings are, run the following commands:

```bash
gcloud config get-value compute/zone
gcloud config get-value compute/region
```

> If the google-compute-default-region or google-compute-default-zone responses are (unset), that means no default zone or region is set.

## Identify the default region and zone

In Cloud Shell, run the following gcloud command, replacing <your_project_ID> with the project ID you copied:

```bash
gcloud compute project-info describe --project <your_project_ID>
```

Find the default zone and region metadata values in the output. You'll use the zone (`google-compute-default-zone`) from the output later in this lab.

> If the google-compute-default-region and google-compute-default-zone keys and values are missing from the output, no default zone or region is set.

## Set environment variables

Environment variables define your environment and help save time when you write scripts that contain APIs or executables.

Create an environment variable to store your Project ID, replacing <your_project_ID> with the value for name from the gcloud compute project-info describe command you ran earlier:

```bash
export PROJECT_ID=<your_project_ID>
```
Create an environment variable to store your Zone, replacing <your_zone> with the value for zone from the `gcloud compute project-info describe` command you ran earlier:

```bash
export ZONE=<your_zone>
```
To verify that your variables were set properly, run the following commands:

```bash
echo $PROJECT_ID
echo $ZONE
```
If the variables were set correctly, the echo commands will output your Project ID and Zone.

## Explore gcloud commands
The gcloud tool offers simple usage guidelines that are available by adding the -h flag (for help) onto the end of any gcloud command.

Run the following command:

```bash
gcloud -h
```
You can access more verbose help by appending the --help flag onto a command or running the gcloud help command.

Run the following command:
```bash
gcloud config --help
```
> Note: Press ENTER or the spacebar to scroll through the help content. To exit the content, type Q.

Run the following command:
```bash
gcloud help config
```
The results of the `gcloud config --help` and `gcloud help config` commands are equivalent. Both return long, detailed help.

>Note: Press ENTER or the spacebar to scroll through the help content. To exit the content, type Q.
gcloud Global Flags govern the behavior of commands on a per-invocation level. Flags override any values set in SDK properties.

View the list of configurations in your environment:
```bash
gcloud config list
```
To see all properties and their settings:
```bash
gcloud config list --all
```
List your components:
```bash
gcloud components list
```
This command displays the gcloud components that are ready to use.

## Install a new component
Install a gcloud component that makes working in the gcloud tool easier.

Auto-complete mode

gcloud interactive has auto prompting for commands and flags and displays inline help snippets in the lower section of the pane as the command is typed.

You can use dropdown menus to auto-complete static information, such as command and sub-command names, flag names, and enumerated flag values.

Install the beta components:
```bash
sudo apt-get install google-cloud-sdk
```
Enable the gcloud interactive mode:
```bash
gcloud beta interactive
```
When using the interactive mode, press TAB to complete file path and resource arguments. If a dropdown menu appears, press TAB to move through the list, and press the spacebar to select your choice.

A list of commands is displayed below the Cloud Shell pane. Pressing F2 toggles the active help section to ON or OFF.

To exit from the interactive mode, run the following command:
```bash
exit
```
