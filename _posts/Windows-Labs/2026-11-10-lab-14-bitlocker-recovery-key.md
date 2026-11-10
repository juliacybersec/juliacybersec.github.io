---
layout: post
title: "Lab 14: BitLocker Recovery Key"
---

Lab 14: BitLocker Recovery Key

The ticket

"I left my laptop updating over the weekend. Now it's a blue screen asking for a BitLocker recovery key and I can't get in. I have a meeting in 20 minutes!"

This is one of the most common help desk calls. It usually happens after a BIOS or firmware update, a hardware change, or a drive being moved to another machine.

How BitLocker works

Think of BitLocker as a safe in your house. The combination is held by a trusted doorman, the TPM chip. Every time the computer starts, before handing over the combination, the doorman checks that the house looks the way it did last time: same front door, same locks, same layout (the BIOS/UEFI, boot files and hardware). If everything matches, he opens the safe quietly and you never notice. That's why BitLocker doesn't ask for anything on a normal start-up.

If someone changes the locks (a BIOS update) or carries the safe to a different house (the drive goes into another PC), the doorman refuses. Now you need the spare key: the 48-digit recovery password. Organisations keep it at the "bank", meaning Active Directory, Entra ID/Intune, or the user's Microsoft account.

When does the check happen?

The bouncer check happens before Windows even starts, not when you log on. The TPM checks the startup chain (BIOS/UEFI, boot files) as the machine powers up. If everything matches, it quietly hands over the key, the drive unlocks, and only then do you reach the Windows sign-in screen.

So when a user rings up with the recovery screen, they haven't got as far as their password yet. That's why their password won't help, and why they're stuck.

Why the bouncer is so strict

If a thief pulls the drive out and puts it in their own computer, that's "the door has moved". The TPM in the thief's machine isn't the one holding the key, so the data stays scrambled unless they have the 48-digit recovery key.

Why ask for the Key ID?

The Key ID is the label on the spare key. One person can have several devices, each drive has its own key, and a re-imaged laptop can leave old keys sitting in AD or Entra that no longer work. The help desk matches the first 8 characters of the Key ID on the user's screen to the right key in the list. Read out the wrong one and they just get "incorrect key".

Check who you're talking to first

Handing over a recovery key is like giving someone the key to your front door. You want to be sure they're who they say they are, not some stranger off the street who'll let themselves in and go through everything. And a company laptop is a door into company data and apps, not just one person's files.

The real risk is social engineering. Someone who has stolen a laptop can ring the help desk pretending to be the owner, and the recovery key would undo BitLocker's protection completely. So before reading out a key:

- Verify identity the way your company's policy says: a callback to the number on file, an MFA prompt or verification code, or confirmation from their manager. Don't trust caller ID or a confident voice.
- Check the device in the key list is actually registered to that person.
- Once the key has been used, it should be rotated (replaced with a new one), because it has now been read out over the phone.

Lab setup

My old lab VM ran Windows 11 Home, which doesn't include full BitLocker. Home only has a cut-down "Device encryption" with no BitLocker control panel. So I rebuilt my lab as a Windows 11 Enterprise Evaluation VM in VirtualBox 7.1 (free 90-day evaluation from Microsoft's Evaluation Center), with a virtual TPM 2.0, Secure Boot, and a 64 GB dynamically allocated disk. Most workplaces run Pro or Enterprise anyway, and Enterprise can join a domain, which I'll need for my Active Directory lab.

Part 1: Check the machine

winver confirmed Windows 11 Enterprise Evaluation.
[Screenshot: lab14-01-winver-enterprise.png]

Get-Tpm in an admin Terminal showed whether the virtual TPM was present and ready.
[Screenshot: lab14-02-get-tpm.png]

Then manage-bde -status.
[Screenshot: lab14-03-manage-bde-status.png]

Surprise find: BitLocker was already running

I hadn't turned BitLocker on, but manage-bde -status showed:

Conversion Status: Encryption in Progress
Encryption Method: XTS-AES 128
Protection Status: Protection Off
Lock Status: Unlocked
Key Protectors: None Found

This is Windows 11's automatic device encryption. On a fresh install on hardware with a TPM, newer versions of Windows 11 can start encrypting the drive by themselves during setup.

The important part is Protection Off and Key Protectors: None Found. The data is being scrambled, but nothing is locking it yet. Windows keeps a "clear key" on the drive until a proper key protector (the TPM plus a backed-up recovery key) is set up. In doorman terms, the safe has been built and the valuables are going in, but the combination is on a sticky note stuck to the front of the safe.

Why this matters on a help desk: this is how a lot of people end up on the recovery screen without ever knowing they had BitLocker. It got switched on automatically, the key was backed up to a Microsoft account they've forgotten, or nowhere at all, and then something like a BIOS update triggers recovery.

Part 2: Turning protection on

Once encryption reached 100% (Fully Encrypted), Protection was still Off. I went into Control Panel > System and Security > BitLocker Drive Encryption to turn it on, but the wizard won't continue until the recovery key is backed up somewhere it trusts:

- Save to Microsoft account: not available, because the VM uses a local account
- Save to a file: blocked, because the only drive is the one being protected (you can't lock the spare key inside the safe)
- Print: no printer in the VM

This is deliberate, to stop people locking themselves out.

Lesson learned: I took a photo of the key the wizard showed, but when I cancelled and checked with manage-bde -protectors -get C:, it said no key protectors found. The wizard's key was only a preview and was never saved. Always check the spare key actually exists before you rely on it.

So I did it from the command line instead (admin Terminal), spare key first so a backup exists before anything gets locked:

manage-bde -protectors -get C:
Shows the existing key protectors (there were none).
[Screenshot: lab14-07-protectors-get.png]

manage-bde -protectors -add C: -RecoveryPassword
Creates the 48-digit recovery password and shows its ID (the Key ID). I photographed both, outside the VM.
[Screenshot: lab14-08-add-recovery-password.png]

manage-bde -protectors -add C: -TPM
Adds the TPM protector (the doorman).
[Screenshot: lab14-09-add-tpm.png]

manage-bde -protectors -enable C:
Removes the clear key and switches protection on.

manage-bde -status
Protection Status: Protection On
Lock Status: Unlocked
Key Protectors: TPM, Numerical Password
[Screenshot: lab14-10-protection-on.png]

Why does it still say Unlocked? Because Windows is running from this drive. The doorman already checked the startup and handed over the key when the VM booted, so while Windows is running the drive is open for use. Protection On means that if the drive is taken out, or the startup changes, it stays locked until the right key is provided.

Part 3: Forcing the recovery screen

manage-bde -forcerecovery C:
This makes BitLocker act as though the startup has changed, like after a BIOS update. It removes the TPM protector, so the doorman refuses on the next boot.
[Screenshot: lab14-11-forcerecovery.png]

My first restart booted straight into Windows. manage-bde -status showed the TPM protector was still there, so the command hadn't gone through. Running it again worked: Key Protectors showed only Numerical Password, with Protection On. After restarting, the VM stopped at the blue BitLocker recovery screen before Windows loaded.
[Screenshot: lab14-12-recovery-screen.png]

Playing help desk

Matching the Key ID: the recovery screen shows a Key ID. The first 8 characters matched the ID in my photo of the key, so I knew I had the right spare key.

"It won't let me type the key": the recovery screen runs before Windows loads, so there's no mouse and no cursor. You just type. Things to try:
- Use the number keys along the top row, not the number pad (Num Lock is often off at this stage).
- The function keys work as digits: F1 = 1 through F9 = 9, and F10 = 0.
- On a real laptop, plug in an external USB keyboard.
- In VirtualBox, click inside the VM window so it captures the keyboard. Input > Keyboard > Soft Keyboard gives an on-screen keyboard, and changing the VM's USB controller setting can also help.

The divide-by-11 check: the recovery key is 8 groups of 6 digits, typed in order from the first group. BitLocker checks each group as you type it. Every real group is a multiple of 11, so if one digit is wrong the group fails and won't let you move on. Mine got stuck on the 6th group because a 0 in my photo looked like an 8. If a user is stuck on one group, have them read it out slowly and watch for lookalikes (0 and 8, 1 and 7, 6 and 8, 3 and 8). You can check a group by dividing it by 11 on a calculator: a whole number means it's been read correctly.

Part 4: Putting the doorman back

Once the key was accepted, Windows loaded as normal. But force recovery had removed the TPM protector, so without it the VM would ask for the recovery key at every start. In an admin Terminal:

manage-bde -protectors -add C: -TPM
manage-bde -status
Protection On, Key Protectors: TPM and Numerical Password.
[Screenshot: lab14-13-tpm-restored.png]

A final restart went straight through to Windows with no recovery screen, so the doorman is back on duty.

In a workplace, the recovery key would now be rotated (replaced with a new one), because it has been read out over the phone. Tools like Intune can do this automatically.

Help desk checklist: BitLocker recovery screen

1. Stay calm and reassure the user. Their data is safe; it's locked, not lost.
2. Verify their identity using company policy (callback to the number on file, MFA code, or manager confirmation) before reading out any key.
3. Ask for the Key ID shown on the recovery screen.
4. Look up the user's device in AD, Entra ID/Intune, or their Microsoft account, and match the first 8 characters of the Key ID to the right recovery key.
5. Read the key out slowly, group by group. If a group won't go in, check for lookalike digits.
6. If the keyboard won't work, try the top-row numbers, the F1 to F10 keys, or an external USB keyboard.
7. Once they're in, find out what triggered it (BIOS update, hardware change) and make sure the key gets rotated.

What I learned

- BitLocker can be switched on automatically by Windows 11, sometimes without the user knowing, and with a local account it can sit with Protection Off and no key protectors.
- Protection Off means the key is stored in the clear on the drive, so the data is scrambled but anyone could read it.
- Always check the spare key actually exists (manage-bde -protectors -get C:) before relying on it.
- The Key ID is the label that tells you which spare key to use.
- Each group of the recovery key is a multiple of 11, which is how BitLocker catches typos.
- Recovery keys never go in chats, emails or blog posts, and screenshots of them get blurred.
