---
author: <Pankaj> <Bhojwani> <@PankajBhojwani>
created on: <2020-9-9>
last updated: <2020-9-9>
---

# Proto extensions

## Abstract

This spec outlines adding support for proto extensions. This would allow other apps/programs
to add json snippets to our json files, and will be used when we generate settings for our various profiles. 

## Inspiration

### Goal: Allow programs to have a say in the settings for their profiles

Currently, Ubuntu/WSL/Powershell etc are unable to provide any specifications for how they want
their profiles in Terminal to look - only we and the users can modify the profiles. We want to provide
these installations with the functionality to have a say in this, allowing them to specify things like
their icon, their font and so on. However, we want to maintain that the user has final say over all of
these settings. 

## Solution Design

Currently, when we load the settings we perform the following steps (this is a simplified description,
for the full version see the [spec for cascading default + user settings](https://github.com/microsoft/terminal/blob/master/doc/specs/%23754%20-%20Cascading%20Default%20Settings.md)):

1. Generate profiles from the defaults file
2. Generate profiles from the dynamic profile generators
3. Layer the user settings on top of all the profiles created in steps 1 and 2
4. Validate the settings

To allow for installations to add in their snippets of json, I propose the addition of a new step
in between 2 and 3:

1. Generate profiles from the defaults file
2. Generate profiles from the dynamic profile generators
3. **Incorporate the additional provided json stubs** - these stubs could be:
   1. modifications to existing profiles
   2. additions of full profiles
   3. additions of colour schemes
   4. modifications to colour schemes
4. Layer the user settings on top of all the profiles created in steps 1 through 3
5. Validate the settings

### Specifications of the json stubs

As written above, the json stubs could be modifications to existing profiles, additions to full profiles, modifications to
existing colour schemes or additions of colour schemes.

#### Modifications to existing profiles

The main thing to note for modification of existing profiles is that this will only be used for modifying the
default profiles (cmd/powershell) or the dynamically generated profiles. 

For modifications to existing profiles, the json stub would need to indicate which profile it wishes to modify.
Note that currently, we generate a GUID for dynamic profiles using the "initial" name of the profile (i.e. before
any user changes are applied). For example, the "initial" name of a WSL profile is the \<name\> argument to
"wsl.exe -d \<name\>", and the "initial" name of a Powershell profile is something like "Powershell (ARM)" - depending
on the version. Thus, the stub creator could simply use the same uuidv5 GUID generator we do on that name to obtain the
GUID. Then, in the stub they provide the GUID can be used to identify which profile to modify. 

Since dynamic profiles also have a "source" field which we use as matching criteria, the stub would also need to
have that field if it wants to modify a dynamic profile. These are just static strings that we can easily provide. 

We might run into the case where multiple json stubs modify the same profile and so they override each other. For the initial implementation, we
are simply going to apply _all_ the changes. Eventually, we will probably want some sort of hierarchy to determine
an order to which changes are applied.

Here is an example of a json stub that modifies an existing profile (specifically the Azure cloud shell profile):

```js
{
    "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
    "source": "Windows.Terminal.Azure",
    "fontSize": 16,
    "fontWeight": "thin"
}
```

#### Full profile stubs

Technically, full profile stubs do not need to contain anything (they could just be '\{\}'). However we should
have some qualifying minimum criteria before we accept a stub as a full profile. I suggest that we only create
new profile objects from stubs that contain at least the following

* A name (and this name _should not_ match the names of existing profiles as described above, because that would cause us to interpret this profile as modifying an existing one)
* A commandline argument
* A unique GUID - this is so we avoid conflicts if several different creators name their profile the same thing

As in the case of the dynamic profile generator, if we create a profile that did not exist before (i.e. does not
exist in the user settings), we need to add the profile to the user settings file and re-save that file.

Here is an example of a json stub that contains a full profile:

```js
{
    "guid": "{a821ae62-9d4a-3e34-b989-0a998ec283e6}",
    "name": "Cool Profile",
    "comandline": "powershell.exe",
    "antialiasingMode": "aliased",
    "fontWeight": "bold",
    "scrollbarState": "hidden"
}
```

#### Colour schemes

Once again, we could run into the problem of several json stubs wanting to modify the same colour scheme. We
will require a hierarchy here as well. Though since the 'owner' of a colour scheme is a lot more dubious, I wonder
if we should only allow creation of new colour schemes for now and ignore requests to modify existing colour
schemes. 

### Creation and location(s) of the json files

#### For apps installed through Microsoft store (or similar)

For apps that are installed through something like the Microsoft Store, they should add their json file(s) to
an app extension. During our profile generation, we will probe the OS for app extensions of this type that it
knows about and obtain the json files.

#### For apps installed 'traditionally' and third parties/independent users

For apps that are installed 'traditionally', their installer could simply add a json file to a specific local folder
(specified by us) that we will look through every time we generate profiles. This is also where independent users will
add their own json files for Terminal to generate/modify profiles. 

## UI/UX Design

This feature will allow other installations a level of control over how their profiles look in Terminal. For example,
if Ubuntu gets a new icon or a new font they can have those changes be reflected in Terminal users' Ubuntu profiles.

Furthermore, this allows users an easy way to share profiles they have created - instead of needing to modify their
settings file directly they could simply download a json file into a specific folder. 

## Capabilities

### Accessibility

This change should not affect accessibility.

### Security

Opening a profile causes its commandline argument to be automatically run. Thus, if malicious modifications are made
to existing profiles or new profiles with malicious commandline arguments are added, users could be tricked into running
things they do not want to.

### Reliability

This should not affect reliability - most of what its doing is simply layering json which we already do. 

### Compatibility

Again, there should not be any issues here - the user settings can still be layered after this layering for the user
to have the final say. 

### Performance, Power, and Efficiency

Looking through the additional json files could negatively impact startup time. 

## Potential Issues

Cases which would likely be frustrating:

* An installer dumps a _lot_ of json files into the folder which we need to look through

## Future considerations

How would this affect the Settings UI? 

## Resources

N/A