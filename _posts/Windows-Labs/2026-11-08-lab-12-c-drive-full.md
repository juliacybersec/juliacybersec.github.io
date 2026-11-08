Lab 12: My C: Drive Is Full

The ticket

"My computer keeps saying I'm low on disk space, and now I can't save anything or install updates."

Low disk space is one of the most common help desk calls. Windows needs free space to run updates, write temporary files and use the page file, so a full drive causes problems well beyond "I can't save my file."

The analogy

Think of the C: drive as a wardrobe. When it's crammed, nothing new fits, and even everyday jobs become a struggle. The help desk job isn't to buy a bigger wardrobe straight away. It's to find what's taking up the space: old clothes (temporary files), forgotten boxes (Downloads and the Recycle Bin), and things that belong in the garage (large files that should be stored somewhere else).

Why I did this lab

I recently had to clean up my own laptop because its drive was nearly full. I deleted an old virtual machine I no longer needed. This lab gave me the chance to work through the problem the way I would on a user's PC.

The setup

- Windows 11 VM in VirtualBox
- Tools: Settings > System > Storage, File Explorer, Disk Cleanup (cleanmgr), Storage Sense

Step 1: Baseline

Settings > System > Storage showed about 39.5 GB used. Windows itself was the biggest chunk, which is normal, followed by installed apps, then smaller amounts for other users, temporary files and the Recycle Bin.

[Screenshot: Storage page baseline]

Unexpected lesson: thin provisioning

The VM reported a C: drive of about 699 GB, far more than the free space on my actual laptop. That's because the VirtualBox disk is dynamically allocated: it starts small and grows as Windows writes data. Windows inside the VM thinks it has hundreds of gigabytes free, but the real limit is the host's free space. If the VM writes more than the host has left, the host fills up first and the VM can pause or crash.

This is called overcommitting, or thin provisioning. Businesses do the same with virtual servers and cloud storage, which is why monitoring the real storage underneath matters.

Step 2: Planting the space hog

To simulate a forgotten large file, I created a 5 GB dummy file in the Downloads folder from an admin terminal:

fsutil file createnew C:\Users\YourName\Downloads\old_backup.zip 5368709120

fsutil reserves the space without writing real data, so it didn't use up my host's storage.

Used space went from 39.5 GB to 44.7 GB.

[Screenshot: Storage page after the dummy file]

Step 3: Finding the culprit

The extra space appeared under Temporary files, which surprised me at first. Opening the breakdown showed why: Windows counts the Downloads folder as "temporary" on this page. Downloads showed 5 GB, with its box unticked by default.

(Tip: refresh the Storage page after making changes. I didn't at first, and the numbers hadn't updated.)

[Screenshot: Temporary files breakdown showing Downloads]

Why the Downloads box is unticked

Many people treat Downloads like a filing cabinet, holding tax returns, photos and installers they still need. If a user ticks every box on this page and clicks Remove files, their Downloads folder is wiped, and those files don't go to the Recycle Bin. If they also tick Recycle Bin, anything they deleted earlier is gone too. The likely call-back is "Where are my photos?", with no way to recover them without a backup.

How I'd handle it on a real ticket:

1. Open Downloads with the user, in Details view, sorted by size
2. Ask "Do you still need this?" for the biggest files
3. Move anything they want to keep into Documents or onto a backup drive
4. Only then clean up

Step 4: The fix

In File Explorer I opened Downloads, switched to Details view and sorted by Size. old_backup.zip was at the top. I deleted it and emptied the Recycle Bin, then cleared the temporary files.

Used space dropped to 39.1 GB, slightly below the baseline because the Recycle Bin and temp files were cleared as well.

[Screenshot: Storage page after cleanup]

Step 5: Disk Cleanup's hidden button

When Storage settings doesn't free enough, Disk Cleanup goes further:

1. Win + R, type cleanmgr, press Enter, pick C:
2. Click "Clean up system files" (needs admin rights) and pick C: again

This adds system items such as Windows Update Cleanup, which removes old versions of files replaced by updates.

Trade-off: after Windows Update Cleanup, recent updates can no longer be uninstalled. That's usually fine, but avoid running it straight after a big update in case the update causes problems and needs rolling back.

[Screenshot: Disk Cleanup system files list]

What not to delete

Users sometimes find large system items and want them gone:

- WinSxS folder: stores the components Windows needs for updates and repairs. Deleting it manually can break Windows. The safe way to shrink it is Windows Update Cleanup in Disk Cleanup.
- pagefile.sys: the page file Windows uses when RAM fills up. Windows manages it.
- hiberfil.sys: used for hibernation and fast startup. It can be removed by turning off hibernation, but not by deleting the file.

Rule of thumb: if it's a system folder or file, use Windows' own tools to clean it, never manual deletion.

Step 6: Stopping the ticket coming back

Temporary files grow back because Windows writes them all the time, so a one-off cleanup isn't a permanent fix. Storage Sense cleans up on a schedule.

Settings > System > Storage > Storage Sense defaults on my VM:

- Run Storage Sense: during low free disk space
- Delete files in my Recycle Bin: after 30 days
- Delete files in my Downloads folder: never (by default, for the same reason the Downloads box is unticked)

[Screenshot: Storage Sense settings]

Lessons learned

- Check where the space is going before deleting anything. Storage settings shows categories, and Details view sorted by size finds the big files.
- Downloads counts as "temporary files" in Storage settings, but it often holds files people need. Always check with the user first.
- Files removed through the Temporary files page skip the Recycle Bin.
- Refresh the Storage page after changes, or the numbers will be out of date.
- "Clean up system files" in Disk Cleanup frees the most space, but removes the option to uninstall recent updates.
- Never manually delete system folders like WinSxS.
- Storage Sense stops the problem coming back.
- In virtual environments, the space a VM reports isn't necessarily the space that really exists.
