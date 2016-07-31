---
layout: post
title: "TrendMicro CTF 2016 Writeup: BlackJack"
---

Time for another CTF Writeup! My [friends](https://ctftime.org/team/12919) dragged me into it, and I ended up netting 400 points on this blackjack-themed "IoT/Network" challenge (I'd say "programming" would be a more fair classification).

<!--more-->

## Getting Started

Unzipping the challenge archive gave me three files: `CTF_BlackJack_Challenge.pl`, `UnRAR.exe`, and `DecryptMe.rar`. After throwing out `UnRAR.exe` (Really???) I took a peek at the perl script and ran it a few times.

It's a standard blackjack game, with some instructions saying that the goal is to win 27 games in a row, with at least one game won with a "blackjack" (a two-card hand consisting of a jack of clubs or spades and an ace). If you can do this, you'll get the password to decrypt `DecryptMe.rar`.

```perl
	myprint( "[just another game of]                                   28 Sept 2016 GMT\n");
	myprint( "==============================BLACKJACK(21)==============================\n");
	myprint( " In order to win:                                                        \n");
	myprint( " 1. Reach a final score higher than the dealer without exceeding 21.     \n");
	myprint( " 2. Let the dealer draw additional cards until his or her hand exceeds 21\n"); # -Wikipedia
	myprint( " 3. You have to win <" . $TargetWins . "> consecutive times out of <" . $MaxGame . "> games.\n");
	myprint( " 4. 1 of the wins should be a \"blackjack win\".                         \n"); # 1 ACE + 1 Jack of (Spades or clubs)
	myprint( "-------------------------------------------------------------------------\n"); # rule 5: no changing of game logic
```

I checked out what the script will do when you meet its winning conditions. It helpfully calls `UnRAR.exe` for you (I changed this to use UNIX `UnRAR`) using a password which is generated from whatever cards remain in the deck at the end of the game. This immediately gave me a huge hint at how to solve the challenge: there had to be some flaw in the RNG which shuffles the deck leading to a deterministic game with a fixed deck state at the end, so that you'll always get the same password.

I took a look at the shuffle function and found that it uses perl's `rand()` function, which is deterministic for a given seed. It's also only seeded once, using `time()` when the program is started up. I tried replacing the seed with 0 and found that the game was obviously unwinnable: either the dealer would win or you'd go bust. It would take a very unlikely shuffle (thus, seed) to even have a chance at winning 27 games in a row without breaking the secret 5th rule:

```perl
# rule 5: no changing of game logic
```

Then I noticed a mysterious date in the header of the rules text the game prints out:

```perl
	myprint( "...28 Sept 2016 GMT\n");
```
Suspicious indeed that it provided a time zone, but no hour/minute/second. Maybe this was a hint as to what seed we should use? This seemed to fit with the challenge description, which mentions "a blackjack tournament coming soon". If we interpret this as the date of the blackjack tournament, maybe we just need to figure out what time the tournament "started" in order to have a chance of winning 27 games in a row.

## First Attempt

I swapped the `PlayerTurn` subroutine for `PlayerStrategy`, modifying it so that it would do the most obvious winning strategy I could think of: it peeks at the first card of the deck and draws it only if it's safe to do so. In pseudocode:

```
PlayerStrategy:
	peek at top card of deck
	if soft value of (hand + top card) <= 21:
		hit
	else:
		stand
```

Or in perl:

```perl
sub PeekCard
{
       my $PeekDeckIndex = $DeckIndex + 1;
       if($PeekDeckIndex > scalar(@DeckOfCards))
       {
	       print("TRYING TO PEEK BUT CAN'T\n"); # I didn't handle this case, but it never came up.
	       exit();
       }
       return @DeckOfCards[$PeekDeckIndex];
}

sub PlayerStrategy
{
	my @HypotheticalHand = @PlayerHand;
	push(@HypotheticalHand, PeekCard());
	my $HypotheticalHandValue = GetHandValue(@HypotheticalHand);
	while($HypotheticalHandValue <= 21)
	{
		@PlayerHand[scalar @PlayerHand] = GetCard();
		@HypotheticalHand = @PlayerHand;
		push(@HypotheticalHand, PeekCard());
		$HypotheticalHandValue = GetHandValue(@HypotheticalHand);
	}
}
```

Then I changed the script to seed the random number generator with the provided date (28th September 2016) plus some number of seconds (passed in on the command line):

```perl
use Time::Local;

$sec = $ARGV[0];
$time = timegm(0, 0, 0, 28, 8, 116) + sec;

print("Playing a game at " . gmtime($time) . "\n");
ShuffleDeck(srand($time));
```
And commented out the handy `myprint` subroutine to silence all the game interface text.

Now I could try playing a blackjack game for every second of the day on 28 September 2016 thusly:

```bash
seq 90000 | parallel -n 1 ./CTF_BlackJack_Challenge.pl
```

## Aside: Red Herrings & Perl Bashing

While that was running I took a look at the source some more, to see if I was missing anything.

The shuffle function is awful: it doesn't result in a uniform deck (see [this stack overflow answer](http://stackoverflow.com/a/7291502) for a good explanation of why). Significant? probably not; I can't see how this flaw could change things as regards getting a deterministic password at the end.

There're also a couple of great off-by-one bugs in the `GetCard` subroutine. It checks if we've run out of cards thusly:

```perl
	if($DeckIndex > scalar(@DeckOfCards))
```
when it should use `>=`. It's therefore possible to try to draw from beyond the end of the deck. To see if this could mess with the game's logic interestingly, I initialized `$DeckIndex` to `scalar(@DeckOfCards)`. Because perl is a great language, trying to access elements beyond the end of an array returns `undef`, which is treated sorta-kinda-similarly to `0`, so the nonexistent 105th card in the deck is treated like an ace of clubs, if you draw it. Then, the next card you draw will trigger a reshuffle, but because `ShuffleDeck` sets `$DeckIndex` to `-1`, the next card you get will be the new LAST (bottom) card from the deck (which isn't removed and will be drawn again 104 cards later)! Yes, drawing from beyond the end of the array silently yields `undef` but drawing from before the beginning of the array wraps around to the end. Because perl is a great language.

Ultimately none of this matters, because if the deck has fewer than 15 cards at the start of a round, it gets shuffled preemptively. Theoretically you could still hit the buggy code if all 15 cards left in the deck at the start of a round were aces and twos, but it's unlikely and never happened in my experience.

Also, while I'm whining about perl, DID YOU KNOW THAT MONTHS ARE INDEXED FROM 0 IN PERL? Yes, 8 means September. But days are still indexed from 1! And don't even get me started on years!!! (indexed from 1900, except for years over 999, which are indexed from 0, and [IT GETS WORSE](http://perldoc.perl.org/Time/Local.html#Year-Value-Interpretation) for years below 99 and oh god please can I go back to Python now!?)

## Let's Try Again

Ok, so my top-card-peeking-strategy finished running and never managed to win 27 games in a row. This makes sense if you think about it a bit; the strategy I used is optimal if you know only the top card on the deck at any time, but you can do better if you know the entire state of the deck. There might, for instance, be a situation where you can draw a card and get a better hand, but in doing so you set the dealer up to draw an even better one.

Imagine you have 17 and the top card is a 2. Drawing it gives you a better hand. But what if the dealer has 13, and the card after the 2 is a 8? If you draw the 2, you get 19 but can't draw the 8, so the dealer does and gets 21 (you lose). But if you leave the 2, the dealer will draw it, and then (following its fixed strategy) draw again, putting them at 23 (you win).

I contemplated analyzing the entire tree of possible decisions but that sounded really hard. The game uses tons of globals so you can't nicely build a tree and then go back up it to try other branches. But then I remembered that the kernel can do all that nonsense for me! I implemented a new player strategy:

```
PlayerStrategy:
	fork
	if parent process:
		hit
	else:
		stand
```

Or, in my new least-favorite language:

```perl
sub PlayerStrategy
{
	my $PlayerHandValue = GetHandValue(@PlayerHand);
	while($PlayerHandValue <= 21)
	{
	if (fork())
	{
		@PlayerHand[scalar @PlayerHand] = GetCard();
	}
	else
	{
		last;
	}
	$PlayerHandValue = GetHandValue(@PlayerHand);
}
```

In a more complex game (go, say,) this would basically be a forkbomb. But in blackjack, there are relatively few turns, only two possible moves, and most of those moves lead to a speedy (process) death, so this is actually a pretty efficent (and very easy to implement) way to search the entire space of possible blackjack games (for a given random seed) for that mythical 27-game run.

I set this running, and after an hour or so (within minutes of the deadline), found `Decrypted.txt` waiting for me.

## Files

### CTF_BlackJack_Challenge.pl

My modified version of the blackjack game, which can brute-force the solution if run like this:

```bash
# 90000 is a bit more than the number of seconds per day.
# you can use xargs instead of parallel but it'll be slower.
seq 90000 | parallel -n 1 ./CTF_BlackJack_Challenge.pl
```

```perl
#!/usr/bin/perl -w
use Digest::SHA qw(sha512_base64);
use Time::Local;
no warnings;

#===========Global=======
my @DeckOfCards;
my $DeckIndex;
my @BankerHand;
my @PlayerHand;
my $TargetWins    = 27;
my $MaxGame       = 27;
my $RemGame       = $MaxGame; 
my $Wins          = 0;
my $BlackJackWins = 0;
my $History       = ""; 
my $Status        = "";
#========================

sub ShowCard
{
	my $cardv = $_[0];
	my $card = "";
	my $suit = $cardv % 4;
	if   ($suit == 0) { $suit = "-CLU "; }
	elsif($suit == 1) { $suit = "-SPD "; }
	elsif($suit == 2) { $suit = "-HRT "; }
	else              { $suit = "-DIA "; }
	
	$value = int($cardv / 4) + 1;
	if($value == 1)
	{ $card = $card . " A" . $suit; }
	elsif($value == 10)
	{ $card = $card . "10" . $suit; }		
	elsif($value == 11)
	{ $card = $card . " J" . $suit; }
	elsif($value == 12)
	{ $card = $card . " Q" . $suit; }
	elsif($value == 13)
	{ $card = $card . " K" . $suit; }
	else
	{ $card = $card . " " . $value . $suit; }

	return $card
}



sub ShuffleDeck
{
	@DeckOfCards = (0..51,0..51);  # 2 decks
	$DeckIndex = -1;
	my $i = scalar @DeckOfCards;
    while ( --$i )
    {
        my $j = int rand( $i+1 );
		$exchange = @DeckOfCards[$j];
        @DeckOfCards[$j] = @DeckOfCards[$i];
		@DeckOfCards[$i] = $exchange
    }
}

sub GetCard
{
	$DeckIndex = $DeckIndex + 1;
	if($DeckIndex > scalar(@DeckOfCards))
	{ ShuffleDeck(); }
	return @DeckOfCards[$DeckIndex];
}

sub PeekCard
{
	my $PeekDeckIndex = $DeckIndex + 1;
	if($PeekDeckIndex > scalar(@DeckOfCards))
	{
		print("TRYING TO PEEK BUT CAN'T\n");
		exit(1);
	}
	return @DeckOfCards[$PeekDeckIndex];
}

sub GetHandValue
{
	my @cards = @_;
	my $cardnum = scalar @_;
	
	my $ace = 0;
	my $cnt = 0;
	my $totalval = 0;
	
	while($cnt < $cardnum)
	{
		$value = int($cards[$cnt] / 4) + 1;	
		if($value == 1)
		{ $ace = 1; }
		elsif($value > 10)
		{ $totalval = $totalval + 10; }
		else
		{ $totalval = $totalval + $value; }
		$cnt = $cnt + 1;
	}
	
	if($ace == 1)
	{
		if($totalval + 11 <= 21)
		{ $totalval = $totalval + 11; }
		else
		{ $totalval = $totalval + 1; }
	}
	return $totalval;
}

sub DisplayHand
{
	my @cards = @_;
	my $cardnum = scalar @_;
	
	my $cnt = 0;
	my $totalval = 0;
	my $hand = "";
	
	while($cnt < $cardnum)
	{
		$suit = $cards[$cnt] % 4;
		if   ($suit == 0) { $suit = "-CLU "; }
		elsif($suit == 1) { $suit = "-SPD "; }
		elsif($suit == 2) { $suit = "-HRT "; }
		else              { $suit = "-DIA "; }
		
		$value = int($cards[$cnt] / 4) + 1;
		if($value == 1)
		{ $hand = $hand . " A" . $suit; }
		elsif($value == 10)
		{ $hand = $hand . "10" . $suit; }		
		elsif($value == 11)
		{ $hand = $hand . " J" . $suit; }
		elsif($value == 12)
		{ $hand = $hand . " Q" . $suit; }
		elsif($value == 13)
		{ $hand = $hand . " K" . $suit; }
		else
		{ $hand = $hand . " " . $value . $suit; }

		$cnt = $cnt + 1;
	}
	return $hand;
}

sub PlayerMove
{
	my $c;	
	my $PlayerHandValue = GetHandValue(@PlayerHand);
	
	while($PlayerHandValue <= 21)
	{
		DisplayTest(1);
		myprint("> [H]it\n");
		myprint("> [S]tand\n");
		myprint("> E[x]it\n");
		myprint("[:] Player Decision:");
		chomp ($c = <>);
		
		if($c =~ /^h/i)
		{ @PlayerHand[scalar @PlayerHand] = GetCard(); }
		elsif($c =~ /^s/i)
		{ last; }
		elsif($c =~ /^x/i)
		{ exit; }
		else
		{ myprint("I do not understand your command ($c)\n");  sleep(1); }
		
		$PlayerHandValue = GetHandValue(@PlayerHand);
	}
}

sub PlayerStrategy
{
	my $PlayerHandValue = GetHandValue(@PlayerHand);
	while($PlayerHandValue <= 21)
	{
		if (fork())
		{
			@PlayerHand[scalar @PlayerHand] = GetCard();
		}
		else
		{
			last;
		}
		$PlayerHandValue = GetHandValue(@PlayerHand);
	}
}



sub BankStrategy
{
	my $BankHandValue = GetHandValue(@BankerHand);
	while($BankHandValue < 17) #Soft 17
	{
		DisplayTest(0);
		@BankerHand[scalar @BankerHand] = GetCard();
		$BankHandValue = GetHandValue(@BankerHand);
		#sleep(1);
	}
	DisplayTest(0);
}

sub CheckWinner
{
	#returns 
	#  0 = banker winner
	#  1 = player winner
	#  2 = Draw
	$BankHandValue = GetHandValue(@BankerHand);    #GlobalVar
	$PlayerHandValue = GetHandValue(@PlayerHand);  #GlobalVar
	
	if(($BankHandValue > 21) && ($PlayerHandValue <= 21))
	{ 
		$Status = "Player Wins! (Banker Bust)";
		return 1;
	}
	elsif(($BankHandValue <= 21) && ($PlayerHandValue > 21))
	{ 
		$Status = "Banker Wins! (Player Bust)";
		return 0;
	}
	elsif(($BankHandValue > 21) && ($PlayerHandValue > 21))
	{ 
		$Status = "Both Bust..";
		return 2;
	}
	elsif($BankHandValue == $PlayerHandValue)
	{ 
		$Status = "Push..";
		return 2;
	}
	elsif($PlayerHandValue > $BankHandValue)
	{ 
		$Status  = "Player Wins!";
		return 1;
	}
	else
	{
		$Status  = "Banker Wins!";
		return 0;
	}
}

sub GenerateFlag
{
	my $rawFlag = "";
	while($DeckIndex < 104)  # Remaining Cards
	{ 
		$rawFlag = $rawFlag . GetCard(); 
	}
	print("testing key: " . sha512_base64($rawFlag) . "\n");
	my $execme = "unrar e -p" . sha512_base64($rawFlag) . " -y DecryptMe.rar";
	system($execme);
	exit(0);
}

sub DisplayTest
{
	my $limitbanker = $_[0];
	
	system("cls");
	myprint( "[just another game of]                                   28 Sept 2016 GMT\n");
	myprint( "==============================BLACKJACK(21)==============================\n");
	myprint( " In order to win:                                                        \n");
	myprint( " 1. Reach a final score higher than the dealer without exceeding 21.     \n");
	myprint( " 2. Let the dealer draw additional cards until his or her hand exceeds 21\n"); # -Wikipedia
	myprint( " 3. You have to win <" . $TargetWins . "> consecutive times out of <" . $MaxGame . "> games.\n");
	myprint( " 4. 1 of the wins should be a \"blackjack win\".                         \n"); # 1 ACE + 1 Jack of (Spades or clubs)
	myprint( "-------------------------------------------------------------------------\n"); # rule 5: no changing of game logic
	myprint( " Total Wins: $Wins($BlackJackWins) [$RemGame Game(s) Left]                 "); 
	$line1 = " "; $line2 = " "; $line3 = " ";
	
	if($limitbanker > 0)
	{
		myprint( "       Player's Turn\n" );
		myprint( "=========================================================================\n\n");
		myprint("Dealer's Card: (UNK)\n");
		$line1 = $line1 . "--------- ";
		$line2 = $line2 . "| ????? | ";
		$line3 = $line3 . "--------- ";
	}
	else
	{
		myprint( "       Dealer's Move\n" );
		myprint( "=========================================================================\n\n");
		myprint( "Dealer's Card: (" . GetHandValue(@BankerHand) . ")\n");
	}
	while($limitbanker < scalar @BankerHand)
	{
		$line1 = $line1 . "--------- ";
		$line2 = $line2 . "|" . DisplayHand(@BankerHand[$limitbanker]) . "| ";
		$line3 = $line3 . "--------- ";
		$limitbanker = $limitbanker + 1;
	}
	myprint( $line1 . "\n");
	myprint( $line2 . "\n");
	myprint( $line3 . "\n\n");
	
	myprint( "Player's Card: (" . GetHandValue(@PlayerHand) . ")\n");
	$i = 0; $line1 = " "; $line2 = " "; $line3 = " ";
	while($i < scalar @PlayerHand)
	{
		$line1 = $line1 . "--------- ";
		$line2 = $line2 . "|" . DisplayHand(@PlayerHand[$i]) . "| ";
		$line3 = $line3 . "--------- ";
		$i = $i + 1;
	}
	myprint( $line1 . "\n");
	myprint( $line2 . "\n");
	myprint( $line3 . "\n\n");
}

sub myprint
{
#	my $stringtoprint = $_[0];
#	print $stringtoprint;
#	#todo:
}


#-----------------------------------------------

$sec = $ARGV[0];
$time = timegm(0, 0, 0, 28, 8, 116) + $sec;

print("Playing a game at " . gmtime($time) . "\n");
ShuffleDeck(srand($time));
while( $RemGame-- )
{	
	if($DeckIndex > (scalar @DeckOfCards - 15))
	{ ShuffleDeck(); }
	
	@BankerHand = ();
	@BankerHand[0] = GetCard();
	@BankerHand[1] = GetCard();
	@PlayerHand = ();
	@PlayerHand[0] = GetCard();
	@PlayerHand[1] = GetCard();
	
	my $BankerBJ = 0;
	my $PlayerBJ = 0;
	if(((@BankerHand[0] == 40) || (@BankerHand[0] == 41)) && (GetHandValue(@BankerHand) == 21))
	{ $BankerBJ = 1; }
	if(((@PlayerHand[0] == 40) || (@PlayerHand[0] == 41)) && (GetHandValue(@PlayerHand) == 21))
	{ $PlayerBJ = 1; }
	
	if($PlayerBJ && $BankerBJ)
	{ 
		DisplayTest(0);
		myprint("[:] Push.. Both Black Jack!?!\n"); 
		myprint("[:] Lost (not enough remaining games to win)\n");
		last;
	}
	elsif($PlayerBJ)
	{
		$Wins = $Wins + 1;
		$BlackJackWins = $BlackJackWins + 1;
		DisplayTest(0);
		myprint("[:] Player Wins! Black Jack!\n");
	}
	elsif($BankerBJ)
	{
		DisplayTest(0);
		myprint("[:] Banker Wins! Black Jack!\n");
		myprint("[:] You lost...\n");
		last;
	}
	else
	{	
		#PlayerMove();
		PlayerStrategy();
		BankStrategy();

		$Winner = CheckWinner();
		if($Winner == 0)
		{
			myprint("[:] $Status\n");
			last;
		}
		elsif($Winner == 1)
		{ 
			$Wins = $Wins + 1;
			DisplayTest(0);
			myprint("[:] $Status\n");
			if(($Wins >= $TargetWins) && $BlackJackWins)
			{ GenerateFlag(); }
		}
		else
		{
			myprint("[:] $Status (not enough remaining games to win)\n");
			last;
		}
	}
	myprint("[x] Press any key to continue..\n");
	#my $resp = <STDIN>;
}
exit(1);
#----------------------bSy 17.06.2016----------------------
```

### Decrypted.txt

This is what was inside `DecryptMe.rar`:

```
ACCDFISA (a.k.a. Anti Child-Porn Ransomware)
One of the early ransomware that uses 3rd party tools for encryption; specifically, WinRAR. 
In an attempt to recover encrypted files, several teams studied the malware. Rather than looking "brute forcing" WinRAR (which would take years), most concentrated on how the malware generates the password. 
The malware uses a PseudoRandom Generator, with GetTickCount as its seed. (GetTickCount returns the number milliseconds that have elapsed since the system was started). While still billions of possible combinations, this is a lot manageable than the original 50+ characters used by the ransomware.
If the user (or some logs) could give the (1)time the machine was powered on and (2)the time the ransomware was executed (e.g. File CreateTime); the number of possible combinations would drop to less than 200 million. A few hours of password brute-forcing should be enough to succeed.
TMCTF{OurGreatestGloryIsNotInNeverFallingButInRisingEveryTimeWeFall}
```

