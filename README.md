# Wordle-SQL
A nerd's guide to Wordle guess optimization using SQL

Wordle is a fun game. But the way people play it bugs me. They typically try to do it depth-first. For example, suppose your first guess is HELLO. You match the correct position on the H, but no other letters. Next, you might try HAPPY and assume you matched both the H and the A. Then you subsequently guess HALOS, etc.. When you guess HALOS, that's an inefficient search because the word cannot contain L or O, as shown in the first guess. Similarly, HELLO is an inefficient search. Unless you happen to get lucky and match on both L's, then you've wasted a potential letter on the duplicated L, that could have been taken up by another letter. 

For the purposes of this piece, matching the *position* of a letter means that you guessed the letter correctly. Like if the word is HELLO, and you guessed HAPPY, then you guessed the position of the H. If I say you matched on the *occurrence* of a letter, it means your guess contained a correct letter, just not in the right position. For example, if the word is HELLO and you guessed REACH, you guessed the E in the correct position. You also guessed the H, but only its occurrence, because it was in the wrong position.

My preferred method is a breadth-first search initially. Eliminate as many possible letters as you can right from the outset. Then once you have a better understanding of the remaining letters, switch to a depth-first search. The depth-first search, like how a normal human guesses, would take information from previous guesses into account in future guesses.

I obtained the dictionary of English language words from https://github.com/dwyl/english-words?tab=readme-ov-file. I used the words_alpha.txt file since that is only alphabetical words instead of alphanumeric words. "First" and "1st" would be contained in the alphanumeric list, but "1st" would not be in the alphabetical-only list. I loaded the file into SQL to obtain the optimal way to search for the answer.

First, which word should be the first one to search for? I believe it should be the ones with the highest use of high-frequency letters. That is, if RSTLNE are the most-commonly used letters (as Wheel of Fortune would have us believe), then our first word should be composed of as many of letters as possible, without duplication. But I don't inherently believe that those are the most used letters, so let's find out which ones are.

```
SELECT  SUM([A count]) [A count],
		SUM([B count]) [B count],
		SUM([C count]) [C count],
		...
		SUM([X count]) [X count],
		SUM([Y count]) [Y count],
		SUM([Z count]) [Z count]
FROM (
	SELECT [Word], 
			LEN([Word])-LEN(REPLACE([Word], 'a', '')) [A count], 
			LEN([Word])-LEN(REPLACE([Word], 'b', '')) [B count], 
			LEN([Word])-LEN(REPLACE([Word], 'c', '')) [C count], 
			...
			LEN([Word])-LEN(REPLACE([Word], 'x', '')) [X count], 
			LEN([Word])-LEN(REPLACE([Word], 'y', '')) [Y count],
			LEN([Word])-LEN(REPLACE([Word], 'z', '')) [Z count]
	  FROM [dbo].[Dictionary]
	  WHERE LEN([Word]) = 5
) [g]
```

Let's go through this piece by piece. Starting from the inside query, the `WHERE` clause will filter to only words with five letters. The `SELECT` statement finds out how many times each letter appears in the word. It uses the common replacement-length method. If you want to know how many times the letter L appears in HELLO, then replace all instances of L with nothing. The length of HELLO is five, and the length of HEO is three. Therefore, there must be two L's in HELLO (five minus three). Now consider the outer query. The `SUM` statements aggregate the number of occurrences of each letter across all words.

I transposed the results here, sorted by frequency descending:
|Letter|Count|
|------|-----|
|A count|8,393|
|E count|7,803|
|S count|6,537|
|O count|5,219|
|R count|5,145|
|I count|5,067|
|L count|4,247|
|T count|4,189|
|N count|4,044|
|U count|3,361|
|D count|2,813|
|C count|2,745|
|Y count|2,523|
|M count|2,494|
|P count|2,299|
|H count|2,284|
|B count|2,091|
|G count|1,971|
|K count|1,743|
|F count|1,238|
|W count|1,171|
|V count|878|
|Z count|474|
|J count|376|
|X count|361|
|Q count|139|

Now that we have our list of letter rankings, we can arbitrarily assign a score to them. Since there are 26 letters, I'll simply say that the highest frequency letter gives a score of 26, the next frequent a score of 25, etc. Our objective now is to find words with no duplicates letters that have the highest score. And since we're doing a breadth-first search, whatever word we search for in our second guess should share no letters with the first guess.

```
SELECT [Word], 
			((LEN([Word])-LEN(REPLACE([Word], 'a', '')))*26)+  
			((LEN([Word])-LEN(REPLACE([Word], 'b', '')))*10)+  
			((LEN([Word])-LEN(REPLACE([Word], 'c', '')))*15)+  
			...
			((LEN([Word])-LEN(REPLACE([Word], 'x', '')))*2)+ 
			((LEN([Word])-LEN(REPLACE([Word], 'y', '')))*14)+ 
			((LEN([Word])-LEN(REPLACE([Word], 'z', '')))*4) [Score]
	  FROM [dbo].[Dictionary]
	  WHERE LEN([Word]) = 5  
	  AND	   
	  LEN([Word])-LEN(REPLACE([Word], 'a', '')) <= 1 AND
	  LEN([Word])-LEN(REPLACE([Word], 'b', ''))	<= 1 AND
	  LEN([Word])-LEN(REPLACE([Word], 'c', ''))	<= 1 AND
	  ...
	  LEN([Word])-LEN(REPLACE([Word], 'x', ''))	<= 1 AND
	  LEN([Word])-LEN(REPLACE([Word], 'y', ''))	<= 1 AND
	  LEN([Word])-LEN(REPLACE([Word], 'z', ''))	<= 1
	ORDER BY [Score] DESC
```

The new conditions in the `WHERE` clause state that the frequency of each letter must be no more than one. In the `SELECT` statement, I've multiplied each occurrence of a letter by its corresponding score to get a grand total score, and then I ordered the results by the grand total score, highest to lowest.

|Word|Score|
|---|---|
|arose|120|
|oreas|120|
|seora|120|
|serai|118|
|raise|118|
|solea|118|
|osela|118|
|arise|118|
|aries|118|
|...|...|

This makes `arose` our first guess. For the Jan 2nd, 2026 puzzle, it matched correctly on the positions of R and O, and no other letters. Our second guess should share no letters with arose. So, we can simply change the corresponding `WHERE` clause statements for A, R, O, S, and E from `<= 1` to `= 0`.

|Word|Score|
|---|---|
|unlit|95|
|until|95|
|clint|93|
|culti|92|
|linty|92|
|unlid|92|
|...|...|

`Unlit` *would be* our second guess, except that there are no words that exclude A, R, O, S, E, U, N, L, I, and T. So, we can instead use `clint` as the second word. It matched on no letters.

|Word|Score|
|---|---|
|dumpy|72|
|dumby|70|
|dumky|68|
|pudgy|68|
|humpy|67|
|budgy|66|
|...|...|

These are all funny words, but I’ll go with dumpy, since it introduces P and Y while avoiding all prior letters. It matched on the occurrence of a P in the fourth position (meaning that the fourth letter is not a P, but a P does appear somewhere in the word). There are no words that exclude all 15 that we've used so far, so let's move to a depth-first search. From the information we have so far, we know that:

* The word is not AROSE, CLINT, or DUMPY
* R, and O are the second and third letters.
* There is at least one P somewhere in there, and it is not the fourth letter.
* The letters A, S, E, C, L, I, N, T, D, U, M, and Y do not appear at all.

We can entirely remove the condition that letters have to appear only once. This changes the `WHERE` clause to:

```
WHERE LEN([Word]) = 5 AND
		[Word] <> 'arose' AND
		[Word] <> 'clint' AND
		[Word] <> 'dumpy' AND
		SUBSTRING([Word], 2, 2) = 'ro' AND 
		LEN([Word])-LEN(REPLACE([Word], 'p', '')) >= 1 AND 
		SUBSTRING([Word], 4, 1) <> 'p' AND
		LEN([Word])-LEN(REPLACE([Word], 'a', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 's', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'e', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'c', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'l', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'i', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'n', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 't', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'd', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'u', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'm', '')) = 0 AND
		LEN([Word])-LEN(REPLACE([Word], 'y', '')) = 0
```

|Word|Score|
|---|---|
|groop|89|
|proof|87|
|...|...|

I’m not sure what groop means (and in any case Wordle would never choose it0 so I’ll go with proof.

And sure enough, PROOF is the correct answer. I know what you're asking. You're asking, "Greg, what if we had simply stuck with a depth-first search from the beginning? I tried it on today's puzzle, and it took 5 guesses instead of 4. And as we all know, a sample size of one is sufficient evidence that it applies to all cases.

Enjoy!
