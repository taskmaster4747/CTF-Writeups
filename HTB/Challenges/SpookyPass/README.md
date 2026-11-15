# Hack The Box Write-up: SpookyPass

Category: Reverse Engineering  
Challenge: SpookyPass  
Difficulty: Easy  
Tools Used: Linux Terminal, `strings`

 1. Challenge Description

Blood: See who has claimed the blood for this challenge.

Scenario:

All the coolest ghosts in town are going to a Haunted Houseparty. Can you prove you deserve to get in?

The objective is to analyze the provided executable and discover the password required to enter the party.

 2. Initial Reconnaissance

After downloading and extracting the challenge files, I navigated to the challenge directory:

 bash
cd /media/sf_HTB/rev_spookypass

I listed the available files:

 bash
 ls


Output:

s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
pass


The directory contained a single executable named `pass`.

 3. Static Analysis

Since this was a reverse-engineering challenge, I decided to inspect the executable without running it first.

I used the Linux `strings` command:

bash
strings pass


The `strings` utility extracts readable text sequences from binary files.

Among the output, I discovered several interesting strings:


Welcome to the
SPOOKIEST
party of the year.

Before we let you in, you'll need to give us the password:

s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5

Welcome inside!

You're not a real ghost; clear off!


 4. Password Discovery

The extracted strings revealed a suspicious string:

s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5


Based on the surrounding messages, I identified this as the password expected by the executable.

The program also contained the following messages:

 `Welcome inside!` — Successful authentication.
 `You're not a real ghost; clear off!` — Failed authentication.

This indicated that the program likely compares the supplied password against a hardcoded value.

 5. Solution

The discovered password was:

s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5

Submitting this password to the challenge provided the required solution.

 6. What I Learned

Through this challenge, I learned:

    1. How to inspect executable binaries using Linux utilities.
    2. How the `strings` command can reveal readable information embedded inside binaries.
    3. How hardcoded credentials can be exposed through static analysis.
    4. How program messages can provide clues about authentication logic.

 7. Conclusion

This challenge demonstrated that sensitive information embedded directly inside executable files can sometimes be recovered without executing the program or performing complex reverse engineering.

Using a simple static-analysis technique with `strings`, I successfully discovered the password and solved the SpookyPass challenge.

Flag / Password:

" s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5 "