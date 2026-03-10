# CTF Writeups

A collection of writeups for CTF challenges I've solved.

| Challenge | Category | Difficulty | CTF |
|-----------|----------|------------|-----|
| [Fancy Restaurant](./cyberhero-playground/fancy-restaurant) | Binary Exploitation | Beginner | CyberHero |

# Fancy Restaurant — CTF Writeup

**Category:** Binary Exploitation

Hi there. Today I solved this PwN CTF called Fancy Restaurant. This is my writeup on how I did it. I will include all mistakes I've made through process at the end :)

The connection was through nc to some server on port 9010. No source code at all, just the compiled ELF binary for Linux. So at the moment the goal seemed straightforward, get the flag somehow :| ???

First thing that crossed my mind is to actually start checking the binary itself. Ofcourse, since there is no code, I figured looking at strings inside it might help me. You know, those plain text bits that C programs leave in like messages or whatever. I dumped the hex and spotted a bunch right away. Things like "WELCOME TO THE FANCY RESTAURANT" and Access is allowed only to elegant people. 

But more important were stats. There was Energy and Weight, both shown with "%hu". So after some time I figured out that means unsigned short integer, soooo 16 bit numbers up to 65535.

Also, the menu options popped out with 4 options. First is to try to enter, second is to eat something, third is go jogging (ok bro I get it, summer is coming xd), fourth is to give up (which I never do l0l). And there was this "print_flag" string, which had to be the way in for sure br0. From that point I pieced together somehow its some game where you manage energy and weight to get into this what they call fancy restaurant. Elegant probably means low weight, maybe zero or something close to it.

Connecting with nc showed the starting values, energy at 5, weight is set 80. So I tried option 1 right away and I got rejected, not elegant enough tho. Makes sense to me, like 80 is heavy I guess. So jogging should drop weight by 1 each time right? That's what I thought. But partly I was actually right. Weight drops down, but...energy drops too, starts low.

My first idea was just j0g 80 times to hit zero weight l0l. But energy only lets you jog like twice or three times before its completely gone. I can't really jog without it so, now what?
I thought, eat to get energy back, then jog more. Wrote a quick python script with pwntools to loop, eat when energy is zero, j0g otherwise.

Ran it and watched, but....weight didn't budge down :((
It kind of bounced between 78 and 80 like forever xd. That was weird tho. Kept going in circles basically.

So I started the script again and did output to tee to analyze it additionally. After a bit, I was when energy hits zero and you eat, sometimes it says you are not hungry and nothing changes ever. But in the loop it was mixing up l000l. Needed to test actions one by one properly. I made another script as you might guess, just send each option separately this time and print the responses. Started with initial connect, then tried 3 for jog. Weight went down to 79, energy to 3, lost 1 kilo message. Then eat with option 2, energy back to 5 but weight jumped to 84, you ate something. Tried again, now it says that I'm not hungry anymore, no change. Jog again, weight 83, energy 3. Eat, weight to 88. Finally man :|

Okay, soo jog is minus 2 energy, minus 1 weight. Eat is plus 2 energy, but...plus 5 weight. Thats way more that I thought. So that way, one jog and one eat nets plus 4 weight total. No wonder it was increasing slowly :/

Stuck for a minute there. And suddenly it hit me. UNSIGNED SHORT FOR WEIGHT ! 16 bits -> max 65535. If somehow I overflow past that, it must wrap to zero in C, right? Like adding 1 to 65535 gives 0. Classic int overflow thing, programs forget to check.

Starting at 80, each cycle adds 4 net. So to reach 65536, wich mods to 0. 

80 + 4 * n = 65536
4 * n = 65456
N = 16364 cycles

So I was at this point very excited cause maybe I figured out this challenge. My new plan was to do exactly 16364 cycles of jog then eat, then try to enter. But looping with recv each time might timeout the server. I tried to do it, and I was right, it closes connections if too slow I bet, but nevermind. Instead, build the whole payload as repeated 

(b'3\n' + b'2\n')  * 16364 + b'1\n'

send it all at once with p.send. Then interactive to see the flag.

Finnaly ran the script. It connected, sent the big payload. I was really nervous but then.

BOOM! It said you may now enter the restaurant, and the flag message. SCC{...}

Mistakes and fails :
Earlier attempts faild hard though. First one just looped forever with no progress at all. Tried spamming eat, but it blocks when not hungry. Another time I just looped per action and as you might guess server just quit :(.
Math was off too tbh. I thought eat added less weight but turned out later that I wasn't right.

The strings part was key, figured the whole logic without running much. And testing actions manually saved it, black box style XD. Pwntools mad the exploit part easy, documentation helped alot. Integer overflow lets you bypass the check by wrapping around. Pretty neat how small types like these cause big issues if unchecked.

Thank you for reading. I think thats the main stuff. I hope you find it helpful in some way. Some people might miss the overflow and keep trying to subtract, but yeah. Tools like strings command help a ton for rev without any decompling. Happy hacking <3
