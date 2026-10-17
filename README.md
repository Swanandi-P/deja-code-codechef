Déjà Code: finding CodeChef problems that share the same idea

Imagine a monk climbing a huge staircase to a temple door. He can take one step or two at a time, and he has to count the ways to reach the top. Now imagine a plain problem that just says: "You can climb 1 or 2 stairs at a time. How many ways to climb N stairs?"

Same problem. Totally different story.

That's the whole task. A good search tool should know those two are the same, even though they share almost no words. This project is my attempt at building one for CodeChef problems, and, just as important, my attempt at being honest about where it works and where it doesn't.

WHAT DOES THIS PROJECT DO?

Takes 1,424 CodeChef problem statements and turns each one into numbers a computer can compare.
Lets you paste any problem statement and returns the 5 most similar problems.
Groups all problems into 12 clusters, plots them in 2D, and gives each cluster an automatic label.
Evaluates the search two different ways, and then looks at where it fails.

Everything lives in one notebook: deja_code.ipynb. It runs top to bottom.

HOW DO I RUN IT?

Install the libraries: py -m pip install -r requirements.txt
Put questions.csv next to the notebook (it's in this repo, and it's from this Kaggle dataset).
Open deja_code.ipynb in Jupyter and click Kernel → Restart & Run All. It takes about 10 minutes the first time, because it encodes every problem twice. After that the embeddings are cached.
To search, use the function near the end of section 7:
python
find_similar("Given a tree, find the diameter, the longest path between any two nodes.", k=5, mode="hybrid")

mode can be "semantic" (meaning), "tfidf" (keywords) or "hybrid" (both).

WHAT DATA DID I USE?

Only questions.csv from the Kaggle dataset: 1,474 problems with title, statement, difficulty, tags and author.

The dataset also has solutions.csv and about 1.2 GB of code files. I deliberately didn't use them. solutions.csv is only a log of submissions with no code in it, and the code files only cover 339 problems that have an accepted solution. That's about a quarter of the problems, so building on them would have left most of the problems out.

HOW DID I CLEAN IT?

Look, raw problem statements are messy. Here's what I did and why:

Removed boilerplate. Every statement starts with "All submissions for this problem are available." That's noise, and it would make every problem look a little bit similar.
Cut off the sample input and output. Numbers in an example say nothing about the idea behind a problem.
Removed duplicates. 1,474 became 1,430, so a problem can't match its own copy.
Dropped statements under 200 characters. They're broken or trivial. That leaves 1,424 problems.
Cleaned the tags. The raw tags are mostly author names and difficulty words, so I mapped them to real topics (graphs, strings, dp and so on) and merged duplicates like dp and dynamic-prog. 620 of the 1,424 problems end up with a topic tag.
A second cleaning pass for the embeddings. I removed translation lines ("this problem is also available in Hindi"), recurring character names, and story sentences, so the model sees more of the problem and less of the story.
Removed scraped-code statements. A few statements are actually pasted web page code (for example "Avantika"). Those are excluded from search results.
HOW DOES THE SEARCH WORK?

I tried three ideas, so I could compare them.

1. TF-IDF (keyword search). It counts words and gives more weight to rare ones. It finds problems that use the same words. It's fast and simple, and it can't tell that "stairs" and "steps on a ladder" mean the same thing.

2. Sentence-BERT embeddings (meaning search). I used the all-MiniLM-L6-v2 model, which turns each statement into 384 numbers so that statements with similar meaning land close together. Similarity is the cosine between two vectors. (It reads up to 512 tokens, so very long statements are cut off at the end.)

3. Centering, then a hybrid.

Centering: embeddings have a problem called "hubness", where a few problems show up as neighbours of almost everything. Subtracting the average embedding from all of them and re-normalising reduces that.
Hybrid: score = 0.5 × meaning score + 0.5 × keyword score, with both scaled to 0–1. The 0.5 is my design choice and I haven't tuned it. I didn't tune it because I only have 18 hand-marked queries, and tuning on those would just be fitting the test.
HOW DID I CHECK IF IT WORKS?

Two ways, because neither one is perfect.

A. Topic tags (automatic, 474 problems with a clear topic). If I search with a problem and the top 5 results share its topic tag, that's a hit. This is big but crude: two problems can share an idea without sharing a tag, and the reverse. A random guess scores 0.054.

B. A hand-marked query set (small, but closer to the real task). I wrote 20 one-line queries (like "Count the palindromic substrings of a string") and read the top-5 results of every method in full. A result counts as relevant only if it asks for the same task (or a harder variant of it). Sharing a topic like "graph" doesn't count. I dropped 2 queries I worded badly, which leaves 18 queries. I marked relevance by reading the top-5 results of two methods (TF-IDF and the final system), so those two are in the answer key. A relevant problem that only another method found, and neither of those two did, would be missing from the key, which means the other three rows are lower bounds.

Results: topic tags
Method	Precision@5	MRR	Same-author neighbours	Max times one problem is in a top-5	Problems never in any top-5
Random guess	0.054				
TF-IDF	0.125	0.270	0.175	44	116
Embeddings (original text)	0.154	0.316	0.138	42	217
Embeddings (cleaned)	0.123	0.258	0.097	45	196
Embeddings (cleaned + centered)	0.132	0.268	0.098	26	78
Results: my 18 hand-marked queries
Method	Precision@5	95% range	Queries with at least one hit (of 18)
TF-IDF	0.111	0.04 to 0.20	7
Embeddings (original)	0.156	0.09 to 0.22	11
Embeddings (cleaned)	0.211	0.13 to 0.30	12
Embeddings (cleaned + centered)	0.200	0.11 to 0.29	11
Hybrid (centered + TF-IDF)	0.156	0.09 to 0.23	11

The 95% ranges come from bootstrapping the 18 queries.

So what do these numbers actually say?

I want to be straight about this, because the numbers are small:

Meaning-based search beats keyword search on my hand-marked queries (12 of 18 queries found something, against 7 for TF-IDF), and the paired difference against TF-IDF for cleaned embeddings is +0.02 to +0.19. That's a real signal.
But the ranges overlap a lot, and with 18 queries I can't claim any method is clearly the winner.
On topic tags, the original embeddings score best, and cleaning made P@5 go down, not up. Cleaning did what I wanted in other ways (fewer same-author matches, 0.138 → 0.097), so I think some of the original score comes from author writing style, not from ideas. That's a hypothesis supported by the numbers (same-author share and cluster agreement both moved), not something I proved.
TF-IDF also has a high same-author share (0.175), so author style leaks into plain word matching too.
Centering helped with hubs. The most over-represented problem dropped from 45 appearances to 26, and problems that never show up in any top-5 dropped from 196 to 78.
The hybrid is my default because it combines both signals, not because I proved it's best. On my 18 queries the cleaned embeddings alone did slightly better.
WHAT DO THE CLUSTERS LOOK LIKE?

I used KMeans with k = 12, then UMAP to squash the embeddings down to a 2D picture (clusters_v2.png). Each cluster is labelled automatically using the words that are most distinctive for it compared to the other clusters:

Cluster	Label	Cluster	Label
0	cell, cells, matrix	6	game, player, players
1	time, minutes, server	7	string, strings, letters
2	points, polygon, triangle	8	array, elements, ai
3	query, queries, array	9	cities, city, road
4	tree, nodes, graph	10	digits, positive, test
5	dishes, dish, items	11	cell, game, cells

The clean ones: strings (7), geometry (2), trees and graphs (4), queries (3), games (6), roads and cities (9). You can read the idea off the label.

The weak ones: 1, 5, 10 and 11. "time, minutes, server" and "dishes, dish, items" are clusters of story, not idea. A cooking story is a cooking story whether the maths underneath is dp or greedy.

The silhouette score is only about 0.02 to 0.03 for every k I tried, and it stays flat. That means the data doesn't have sharp, well-separated groups. Problems blend into each other, so 12 is a reasonable summary, not a discovery. Cluster agreement with the topic tags (NMI) is 0.200 for the original embeddings and 0.254 for the cleaned ones.

WHAT DOES IT ACTUALLY RETURN?

Here are three real runs of find_similar(query, k=5). I'm showing the good, the half-good and the bad one on purpose.

1. Works well: palindromes. Query: "Given a string, count how many substrings of it are palindromes."

Meaning search	Keyword search	Hybrid
Girl Friend and String Gift	3-Palindromes	3-Palindromes
3-Palindromes	Girl Friend and String Gift	Girl Friend and String Gift
Lucy and Palindromes	Puppy and Palindromes	Puppy and Palindromes
Count palindromes	Codechef Password Recovery	Lucy and Palindromes
Palindromic substrings	Palindrome Palindrome	Palindrome Palindrome

All three modes stay inside palindrome problems, and "Palindromic substrings" and "3-Palindromes" are real substring-palindrome problems. Meaning search is a bit cleaner: keyword search drifts to "Codechef Password Recovery", which is a graph-tagged problem that shares words with the query.

2. Half-works: the stairs query. Query: "Count the number of ways to climb N stairs taking 1 or 2 steps at a time."

Keyword search and the hybrid both rank Climbing Stairs first.
Meaning search never finds it in the top 5. It returns "Stepping Numbers", "Robot Movings" and "Number of Ways", which are about counting or steps in a loose way, but none of them is the stairs problem.

This is the buried-idea failure from the next section, and the hybrid only gets it right because the keywords carry it.

3. Right topic, can't tell if the idea matches: the tree query. Query: "Given a tree, find the diameter, the longest path between any two nodes."

All three modes return tree problems (Tree, Trees Again, Chef and Trees, Path containing K Nodes, Prime Distance On Tree). I didn't check whether any of them is really a tree-diameter problem, so I'm not claiming it is. What I can say is the topic is right. This is the kind of result tag-based evaluation would count as a hit and my hand-marked evaluation would be strict about.

WHERE DOES IT FAIL?

This is the part I think matters most.

1. Embeddings find topics, not techniques. Search works well when the query is about what the problem is about: strings, geometry, segment trees, trees and graphs. It does much worse at "the same trick", like binary search on the answer, XOR tricks or expected value. Even the best method found at least one relevant result for only 12 of my 18 queries. For 4 of the queries I couldn't find any relevant problem in anything the methods returned: the cheapest-items-on-a-budget greedy query, polygon area, the expected value of dice rolls, and Fibonacci modulo a prime. Three of those are techniques; polygon area is a plain geometry task, so this isn't only about techniques. Also, the 4 empty queries may have a relevant problem that none of the methods surfaced at all.

2. A long story buries the idea. The monk problem from the top of this README is real: it's CodeChef's "Climbing Stairs", and a long monk-and-temple story hides a 1-or-2-steps recurrence in the middle. For my stairs query, TF-IDF ranked it 1st (the words "stairs" and "steps" are in there), while the original embeddings ranked it 225th, cleaned 134th, and cleaned + centered 98th. The hybrid ranked it 1st, but only because the keywords carried it.

3. Story words dominate. I tested this with a frog-jumping problem. All three modes failed. Semantic search returned problems about "Chef and Frogs" and a pet, and TF-IDF and the hybrid locked onto the word "stones". The model matched the story, and the idea behind it was lost.

4. Hubs. Some problems appear as neighbours of everything. Centering cut this a lot (45 → 26), but didn't remove it.

5. Same story, different problem. Two problems about restaurants and dishes can be matched because of the restaurant story even if the maths is different. That's the clusters-of-story problem from above, showing up in search.

6. Truncated and broken data. A few statements in the raw data are cut off mid-sentence (for example "Greedy Trip") or are scraped web code. I found these late, excluded the scraped-code ones from search results, and can't fix the truncated ones.

LIMITATIONS
Small evaluation. 18 hand-marked queries, written and judged by one person (me). The ranges above are wide, and nothing here proves a winner.
Judgment calls in the answer key. "Relevant" means the same task or a harder variant. Someone else might have drawn that line differently. I also corrected my own marking in places once I read the full statements.
Tags are a crude yardstick. They exist for only 620 of 1,424 problems, and I only used the 474 with a sharp topic.
One small model. all-MiniLM-L6-v2 is fast but small, and it reads at most 512 tokens, so long statements are cut off at the end. I tried a bigger model (mpnet) but didn't finish that run, so I don't have a comparison.
The hybrid weight (0.5) isn't tuned. See above for why.
No interactive app. Search is a function in the notebook, not a web page.
Not tested on a second dataset. Everything is CodeChef problems.

If I had more time, the first thing I'd try is embedding each statement in chunks and scoring by the best-matching chunk, to fix the buried-idea problem.

BONUS ITEMS THIS COVERS
Keyword vs semantic comparison: yes, side by side in sections 5 to 9.
Hybrid search: yes, mode="hybrid".
Same algorithm, different storyline: the Climbing Stairs and frog examples, including where they fail.
Auto cluster labels: yes.
Recommendations ("problems like this one"): find_similar does this.
Interactive interface: no.
FILES
deja_code.ipynb: the whole pipeline, with outputs.
questions.csv: the problem statements (from Kaggle).
clusters_v2.png: the labelled cluster plot.
results_tags.csv, results_hand_marked.csv: the evaluation tables.
clustered_questions_final.csv: every problem with its cluster and label.
requirements.txt: libraries.
ONE LAST THING

Go back to the monk. A good version of this system looks at that long temple story and the plain "1 or 2 stairs" problem and says: same thing. Mine does that sometimes, mostly when words help it, and it fails when the story is loud and the idea is quiet. I'd rather tell you that clearly than show you one cherry-picked example. The story is easy to fool a computer with, so the next step is teaching it to look through the story.
