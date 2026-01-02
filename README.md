# Wordle-SQL
A nerd's guide to Wordle guess optimization using SQL

Wordle is a fun game. But the way people play it bugs me. They typically try to do it depth-first. E.g., let's say that your first guess is HELLO, and you match the correct position on the H (but not any other letters). Next, you might go with HAPPY, and assume you match on the positions of both the H and A. Then you subsequently guess HALOS, etc.. When you guess HALOS, that's an inefficient search because you've already been shown that the word cannot contain L or O, as shown in the first guess. Similarly, HELLO is an inefficient search. Unless you happen to get lucky and match on both L's, then you've wasted a potential letter on the duplicated L, that could have been taken up by another letter.

My preferred method is a breadth-first search; eliminate as many possible letters as you can right from the outset.

I obtained the dictionary of English language words from https://github.com/dwyl/english-words?tab=readme-ov-file. I used the words_alpha.txt file since that is only alphabetical words instead of alphanumeric words. "First" and "1st" would be contained in the alphanumeric list, but "1st" would not be in the alphabetical-only list. I loaded the file into SQL to obtain the optimal way to search for the answer. As far as I am concerned, there are two ways to approach this problem; open-loop and closed-loop. An open-loop approach would be to use SQL to generate a list of very good words to search for to narrow down your list of possible letters as much as possible in the fewest number of guesses. The closed-loop (feedback-driven) way would be to make the first guess, feed the resulting position/frequency information back into your SQL query, and then re-run the query to get the next best guess. For now, I will only be focusing on the open-loop approach, but in the future I may update it to include the closed-loop approach.

First, which word should be the first one to search for? I believe it should be the highest use of high-frequency letters. That is, if RSTLNE are the most-commonly used letters (as Wheel of Fortune would have us believe), then our first word should be composed of as many of letters as possible, without duplication. But I don't inherently believe that those are the most used letters, so let's find out which ones are.

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

Let's go through this piece by piece. Starting from the inside query, the `WHERE` clause will filter to only words with five letters. The `SELECT` statement finds out how many times each letter appears in the word. It uses the commong replacement-length method. If you want to know how many times the letter L appears in HELLO, then replace all instances of L with blanks. The length of HELLO is five, and the length of HEO is three. Therefore, there must be two L's in HELLO (five minus three). Moving to the outer query, the `SUM` statements will sum the number of occurrences of each letter in each word.

I transposed the results here:
|Letter|Count|
|------|-----|
|A count|8393|
|B count|2091|
|C count|2745|
|D count|2813|
|E count|7803|
|F count|1238|
|G count|1971|
|H count|2284|
|I count|5067|
|J count|376|
|K count|1743|
|L count|4247|
|M count|2494|
|N count|4044|
|O count|5219|
|P count|2299|
|Q count|139|
|R count|5145|
|S count|6537|
|T count|4189|
|U count|3361|
|V count|878|
|W count|1171|
|X count|361|
|Y count|2523|
|Z count|474|

