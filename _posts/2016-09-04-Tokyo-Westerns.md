---
layout: post
title: "Tokyo Westerns 2016 Writeups"
---

"Hey John, shouldn't you post something other than CTF writeups on your blog?"

Shut up. CTFs are fun.

<!--more-->

Here are a few writeups from the problems I solved for [Tokyo Westerns 2016](https://tokyowesterns.github.io/ctf2016/).

# <a name="palindrome">Make a Palindrome!</a>

> Your task is to make a palindrome string by rearranging and concatenating given words.
> 
> ```
> Input Format: N <Word_1> <Word_2> ... <Word_N>
> Answer Format: Rearranged words separated by space.
> Each words contain only lower case alphabet characters.
> 
> Example Input: 3 ab cba c
> Example Answer: ab c cba
> 
> $ nc ppc1.chal.ctf.westerns.tokyo 31111
> ```
> 
> * Time limit is 3 minutes.
> * The maximum number of words is 10.
> * There are 30 cases. You can get flag 1 on case 1. You can get flag 2 on case 30.

This is a super-straightforward programming problem (made me feel like I was in a job interview), so I'll let my code speak for itself:

### palindrome.py

```python
from unittest import TestCase


def remove_each(list):
    return [(list[i], list[:i] + list[i+1:]) for i in range(len(list))]


def is_palindrome(string):
    return string == string[::-1]


def palindrome_ordering(words, beginning=[], ending=[]):
    beginning_str, ending_str = "".join(beginning), "".join(ending)

    if not is_palindrome(beginning_str[:len(ending_str)] + ending_str[-len(beginning_str):]):
        return None

    if len(words) <= 1:
        answer = beginning + words + ending
        if is_palindrome("".join(answer)):
            return answer
    else:
        for word, remaining_words in remove_each(words):
            if len(beginning) <= len(ending):
                result = palindrome_ordering(remaining_words, beginning + [word], ending)
            else:
                result = palindrome_ordering(remaining_words, beginning, [word] + ending)
            if result is not None:
                return result


class PalindromeTests(TestCase):

    def test_is_palindrome(self):
        self.assertTrue(is_palindrome("abba"))
        self.assertFalse(is_palindrome("abab"))
        self.assertTrue(is_palindrome(""))
        self.assertTrue(is_palindrome("aba"))

    def test_empty(self):
        challenge = []
        solution = []
        self.assertListEqual(palindrome_ordering(challenge), solution)

    def test_single(self):
        challenge = ['abba']
        solution = ['abba']
        self.assertListEqual(palindrome_ordering(challenge), solution)

    def test_single_impossible(self):
        challenge = ['baba']
        self.assertIsNone(palindrome_ordering(challenge))

    def test_trivial(self):
        challenge = ['a', 'a', 'c']
        solution = ['a', 'c', 'a']
        self.assertListEqual(palindrome_ordering(challenge), solution)

    def test_example(self):
        challenge = ['ab', 'cba', 'c']
        solution = ['ab', 'c', 'cba']
        self.assertListEqual(palindrome_ordering(challenge), solution)

    def test_one(self):
        challenge = ['m', 'b', 'amtraj', 'bja', 'rtmamhcch']
        solution = ['bja', 'rtmamhcch', 'm', 'amtraj', 'b']
        self.assertListEqual(palindrome_ordering(challenge), solution)

    def test_two(self):
        challenge = ['kntzdjlkndhy', 'dztnko', 'lllo', 'hdnklj', 'y']
        solution = ['y', 'hdnklj', 'dztnko', 'lllo', 'kntzdjlkndhy']
        self.assertListEqual(palindrome_ordering(challenge), solution)

if __name__ == '__main__':
    import re
    import sys
    for line in sys.stdin:
        match = re.match(r'Input: \d+((?: \w+)+)', line)
        if match:
            words = match.group(1)[1:].split(' ')
            print(" ".join(palindrome_ordering(words)))
```

### palindrome.sh

```bash
#!/usr/bin/env bash

mkfifo fifo1
mkfifo fifo2

python -u palindrome.py <fifo1 | tee fifo2 & nc ppc1.chal.ctf.westerns.tokyo 31111 <fifo2 | tee fifo1

rm fifo1 fifo2
```

This quickly palindromizes each challenge, and the server gives us not one but two flags!

# <a name="glance">glance</a>

> ![glance.gif]({{ site.url }}/assets/glance.gif)
> 
> I saw this through a gap of the door on a train.

It's an animated gif that's just 2 pixels wide. It looks like what you might see through a narrow gap in a train door, as the problem text implies. We should reconstruct the scene outside the train by separating out the frames of the animation and assembling them side-by-side:

### glance.sh

```
#!/usr/bin/env bash

# Requires macOS (for 'open'), ffmpeg, and ImageMagick

# Break apart the frames with ffmpeg
mkdir frames
ffmpeg -i glance.gif frames/frame_%05d.png

# Use ImageMagick's montage to assemble the frames
montage frames/*.png -geometry +0+0 -tile `ls -l frames | wc -l`x1 flag.png

rm -r frames

open flag.png
```

This produces an image containing the flag:

![The flag text in front of the Windows XP wallpaper]({{ site.url }}/assets/glance-flag.png)


# <a name="rps-ng">rps-ng</a>

> Last year, you have to win fifty games in a row to get the flag. In this year, you have to win forty games.
> 
> ```
> nc ppc1.chal.ctf.westerns.tokyo 15376
> ```

There was also a download link for `rps-ng.c`, we'll take a look at that in a sec. First I connected to the server and played a bit of rocks-paper-scissors.

```
$ nc ppc1.chal.ctf.westerns.tokyo 15376
Let's janken
Game 1/50 Your win: 0/0
Rock? Paper? Scissors? [RPS]R
Rock-Rock
Draw
Game 2/50 Your win: 0/1
Rock? Paper? Scissors? [RPS]R
Rock-Rock
Draw
Game 3/50 Your win: 0/2
Rock? Paper? Scissors? [RPS]R
Rock-Rock
Draw
Game 4/50 Your win: 0/3
Rock? Paper? Scissors? [RPS]R
Rock-Paper
You lose
Game 5/50 Your win: 0/4
Rock? Paper? Scissors? [RPS]R
Rock-Paper
You lose
Game 6/50 Your win: 0/5
Rock? Paper? Scissors? [RPS]P
Paper-Paper
Draw
Game 7/50 Your win: 0/6
Rock? Paper? Scissors? [RPS]S
Scissors-Paper
You win!!
Game 8/50 Your win: 1/7
Rock? Paper? Scissors? [RPS]
```

Ok, seems like a normal game of rocks-paper-scissors to me. Let's look at the code:

### rps-ng.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

char buf[100];
char *pname[3] = {
  "Rock",
  "Paper",
  "Scissors"
};
int table_initalized = 0;
int table[3][3];
int last;

// create random table
void init_table() {
  int i, j;
  for(i = 0; i < 3; i++) {
    for(j = 0; j < 3; j++) {
      table[i][j] = rand() % 6;
    }
  }
  last = 0;
}

void update_table(int c) {
  table[last][c]++;
  last = c;
}

int next_hand() {
  if(!table_initalized) {
    init_table();
    table_initalized = 1;
  }
  int m = -1;
  int ret = 0;
  int i;
  for(i = 0; i < 3; i++) {
    if(m < table[last][i]) {
      m = table[last][i];
      ret = i;
    }
  }
  return (ret + 1) % 3;
}

int main() {
  int seed;
  int i;
  int win = 0;
  FILE *fp = fopen("/dev/urandom", "r");
  fread(&seed, 4, 1, fp);
  fclose(fp);
  srand(seed);

  puts("Let's janken");
  fflush(stdout);

  for(i = 0; i < 50; i++) {
    printf("Game %d/50 Your win: %d/%d\n", i + 1, win, i);
    printf("Rock? Paper? Scissors? [RPS]");
    fflush(stdout);
    int t;
    while(t = getchar()) {
      if(t == EOF) {
        t = 0;
        break;
      }
      if(t == ' ' || t == '\n' || t == '\r' || t == '\t') continue;
      break;
    }
    if(t == 0) {
      puts("Bye bye");
      fflush(stdout);
      return 0;
    }
    int c;
    switch(t) {
    case 'R':
      c = 0;
      break;
    case 'P':
      c = 1;
      break;
    case 'S':
      c = 2;
      break;
    default:
      puts("Wrong input");
      fflush(stdout);
      return 1;
    }
    int p = next_hand();
    printf("%s-%s\n", pname[c], pname[p]);
    if(p == c) {
      puts("Draw");
    }else if((p+1) % 3 == c) {
      puts("You win!!");
      win++;
    } else {
      puts("You lose");
    }
    usleep(100000);
    update_table(c);
    fflush(stdout);
  }
  if(win >= 40) {
    printf("Congrats!!!!\n");

    fp = fopen("flag.txt", "r");
    fgets(buf, 100, fp);
    puts(buf);
  } else {
    printf("Not enough wins\n");
  }
  fflush(stdout);
  return 0;
}
```

## What does it do?

After reading over the code and "running" it (by hand, with a pen and paper), I got an idea of what it does. It starts with a 3x3 array of ints with each initialized to a random value between 0 and 5. Each row of the grid represents your previous move (0 = rock, 1 = paper, 2 = scissors) and each column represents your potential next move. After each hand, the cell corresponding to your last two moves (`table[last_move][current_move]`) gets incremented. So for instance the value in `table[1][2]` represents the number of times you've played scissors when your previous play was paper (plus the initial 0-5 random value).

To decide what move to play against you, the program looks at the row for your previous move (or just defaults to rock for the first round), and guesses what you will do by picking the column with the largest value in that row. Then it plays the move which will beat you, if you play the move it expects.

## How can we beat it?

We can just simulate this strategy to figure out what the program expects us to do, and therefore what it will do to try to beat us. The only problem is that we don't know the initial values for its strategy table.

Still, we could just initialize our table to 0, and we will do better than even and win most rounds. But we need to win 40/50, not just most. Maybe we would eventually get lucky doing this, but we can do better.

We can start by initializing to zero. But then, every time we lose or draw a round, this tells us something about how the contents of the server's table differ from ours.

We know which row the server is using because it's based on our last move. And, we know which cell had the highest value, from the move the server played. So when we draw or lose, we just update the appropriate cell so that it has the highest value in its row in our table, too. After a little while, our table will start to line up with the server's and we will stop losing or drawing.

Here's what that looks like in code:

### rps-ng.py

```python
table = [[0, 0, 0],
         [0, 0, 0],
         [0, 0, 0]]

names = ['R', 'P', 'S']

last = 0
import sys


while True:
    row = table[last]
    top_i = 0
    top = 0
    for i in range(len(row)):
        if top < row[i]:
            top = row[i]
            top_i = i
    
    move = (top_i - 1) % 3

    print(names[move])

    # get result
    while True:
        line = next(sys.stdin)
        if line.startswith("You win"):
            break
        elif line.startswith("Draw"):
            expected_move = (move-1) % 3  # the move they expected us to make
            expected_move_value = top + 1 if expected_move > top_i else top
            table[last][expected_move] = expected_move_value
            break
        elif line.startswith("You lose"):
            # they expected us to make this move!
            move_value = top + 1 if move > top_i else top
            table[last][move] = move_value
            break

    table[last][move] += 1

    last = move
```

### rps-ng.sh

```bash
#!/usr/bin/env bash

mkfifo fifo1
mkfifo fifo2

python -u rps-ng.py <fifo1 | tee fifo2 & nc ppc1.chal.ctf.westerns.tokyo 15376 <fifo2 | tee fifo1

rm fifo1 fifo2
```

This was able to win 40/50 games, most of the time anyway, causing the server to spit out the flag.