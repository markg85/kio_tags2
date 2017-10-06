# KIO Tags 2, what is it?
KDE (or rather plasma) already has tagging support via Baloo. That reason alone is why this is called "Tags 2".
The Baloo way of supporting tags works well if you use Baloo and have a indexed system.

Baloo however is not for tags speciafically. It can do them and supports is, but you get a whole lot more by just using baloo.

This project wants to allow tags in a transparent way. It just wants to give tag support without the need to run a full fledged file indexing service.

# Goals
* macOS has a really good tag system in finder, the goal is to copy that and make it as good in Dolphin (Plasma's default file browser).
* Tags need to be used for multiple purposes. Another usecase would be tagging multiple folders as one "collection", so for instance you can have multiple download locations. If you tag all those locations and and put them under a collection (like "Downloads") then you basically end up with one massive "virtual folder" that can consists of files stored on any number of drives, shares, or whatever else KIO can support. Another example is the recently created "stash" plugin. It could be fully inplemented with tags.
* Store tag data in a very human readable way (either JSON or YAML or SQLite if there is no other way).
* Use xattr to only identify that a file has a tag. Tag data is stored in the just mentioned JSON or YAML file.

# Design
## Daemon
This project sadly does require a daemon to run. That will be either in KDED or KIOD as a module. The daemon had to be "somewhere" because it will need to open the "database" (JSON or YAML), parse it and do calculations on it. You do not want to do that every single time you do anything with tags. At first with only few tags it won't be much of an issue, but with thousands of tags it will become an issue when the file needs to be re-parsed on every tag change. Therefore it needs to run in a daemon.

## Abstract implementation
As noted before, the tags need to be very abstract to allow for multiple usecases. It's not only stash or dolphin tagging that needs to be possible, but also creating collections of tags and allowing other applications (say like gwenview) to use the very same tags as well.

What i plan in doing is making "tag categories". When you request a tag (or do any action on it) you must provide a "category". The tags will then be loaded for that category. API wise, this could look somewhat like this:
 KIO::Tags2::TagDetails details;
 details.name = "Cats";
 details.description = "Some awesome descriptiong of cats";
 details. ...

 KIO::Tags2::setTag(<category>, <file UDSEntry>, details);

The <category> would for instance be "Dolphin".
This way it can also be used for completely different things. Like a category of "Stash".
The API for all of this still has to be made up, but in very rought lines something like the above is what i'm aiming for.

## Xattr
The tags themselves won't be stored in xattr! We will merelly add a single value to the xattr of a file/folder to only indicate if it has a tag or not (a bool). Of that attribute indicates that a tag "should" be there, it is then fetched from the "tag storage" and presented to the user. This does pose an issue with restoring a backup. Regardless of the tags you have, the xattr will be gone. That's tricky!

## Backup and restore
The tag storage itself is proabbly going to be save (likely with QSaveFile) so your tags would still be here after a restore. You just won't see them because your xattributes are likely gone. It remains to be seen how this might be solved.

## Tag storage
I would prefer (if possible and feasible) to have tags stored in JSON therefore i'm only going to describe how i would store it in that file.
Take the following example.

 file.foo (xattr: "hasTags=true"), inode 8423984
 
 tags.json
 {
   tags: ["Code", "Cats", "Dogs", "etc.."]
   files: [{inode: 8423984, tags: [0, 1]}, {inode: ....}, {etc...}]
 }
 
 Tags could be more descriptive:
 {
   tags: [{Name: "Code", Color: "#ff0000", Description: "Some fancy description"}, {...}, {...}]
 }

Here i would have tagged a file called "file.foo". From a file i would only store the inode number. It could potentially be extended a bit to include the device id as well to make it fairly unique, but along those lines is how i would store it. If an inode is in the tags datastore then it must have at least 1 tag. 0 is not possible, multiple obviously is.

A tag itself can have some more information then just the name. In the above example it can also have a color and description, but it should be able to have an image (icon) as well. More is to be made up.
In this design a tag itself cannot be in another tag (which would give you a tag hierarchy). Perhaps sometime in the future.

### INODE, why?
It has a few really nice advantages!
* The tag is persistend across file moves, renames and data changes! The tag even remains if your mount point changes! This means no costly fault sensitive file monitoring for for those reasons. More on "data changes" later on.
* It's just a number (granted, stored as a string in json) but it's a lot shorter then mostly any file URI that would otherwise be stored. This keeps the file small while still allowing for alot of tags. Sure, if you have millions of tags then this porbably isn't a suitable way to store anymore and a SQLite database would be a better fit.
* It allows to easily build up a performant hash table from int -> tag. (small argument. string -> tag is likely sufficiently fast as well).

The above likely works just fine for client side applications that use tags and know the files. They have access to the UDSEntry thus can easily get the inode number and use that to ask for the tags that belong with it. So for applications like Dolphin or Gwenview, this would be sufficient and likely very performant.

But there is a nasty downside as well. As far as i know there is no way to to get the stat of a file when only the inode is know. There is just no function for that. A result of this is that the "tags2://" KIO slave (more on that down below) is not able to fetch the files that belong to a tag. That is obviously unacceptable and would make tags rather useless. So i have to figure out a way to do this while maintaining the advantages of using an inode. A possible way could be to store file paths as well in a compressed binary blob, that remians to be seen.

### Monitoring
As said in the above INODE section, monitoring is not needed for the most operations, that is great! Sadly monitoring is still required because of file removals. The coresponding inode has to be removed as well.
However, the initial version will not monitor for deletion either. Deleted files are not a big issue in the tag datastore even though it polutes it slightly. When you request tags from a file you already know the file is there so for that case there is no need to monitor for deleted files. The only case would be for browsing tags (and their files), there you do not want to see files that don't exist anymore. But because a tag only knows the inode of a file, the application browsing the files would have to stat every file to get it's data anyhow. That code path can be used to exclude deleted files thus you would only see files within a tag that are alive! That would be a very neat solution which would prevent from any file monitoring to be needed!

Therefore initially there is no file monitoring. We'il see how that turns out. If needed, it gets added.

### KIO slave
You would probable like to browse the tags and the files within them. For that reason there is the KIO slave under "tags2://". It allows you to acces just that.
This slave also allows you to reorganize the tags and content within them.

# Todo
All of the above :)
