- Start Date: 2017-07-23
- RFC PR: (leave this empty)
- Yarn Issue: (leave this empty)

# Summary

The main goal of this feature would be removing multiple packages at once and with only one line of command. Instead of using same command multiple times to achieve the same result.

# Motivation

The motivation is very simple -> to reduce time spend typing same command multiple times because if you have to delete only one or two packages, the current command, yarn remove, is sufficient but if you have to delete more packages the time sums up and also this typing will become exhaustive and in addition error prone. And I think that the developers can spend more time on bug fixing than on removing packages. And at last but not least, we can make Yarn even more attractive by giving users more freedom with usage of commands.

# Detailed design

Enhance yarn remove for removing multiple packages. Syntax would be : yarn remove [packages]. So if you just type one package, it will be removed. But if you type for example five packages, the algorithm will iterate through the array of packages and remove them one by one. For example if I want to delete packages : A, B, C, D and E, I will use yarn remove [A,B,C,D,E].

# How We Teach This

Yes, the documentation must be changed to explain new approach. The users can use remove as it is or they can use new version of command

Through documentation and in examples

# Drawbacks

The documentation must be updated, at first syntax can be strange to those users, who do not have programming background;

# Alternatives

To use this syntax : yarn remove A,B,C,D,E instead of yarn remove [A,B,C,D,E]

# Unresolved questions

Which syntax would be better?