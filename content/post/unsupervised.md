---
title: "Trying to organize my Twitter timeline, using unsupervised learning"
date: 2017-11-15T05:45:31-08:00
draft: false
aliases:
    - /blog/4
---

<p>I'm a frequent user of Twitter, but I realize that among the major social networks it could be the hardest to get into. One of the big obstacles for me was that, as I followed more and more people representing my different interests, my timeline became an overcrowded mess with too many different types of content. For example, at one point I started following many tech-related accounts and comic book art-related accounts at the same time, and when I would go on Twitter I could never reasonably choose to consume content from only one of the groups.</p>
<figure>
  <img src="https://i.imgur.com/OtK3Yu6.png" width="500" />
  <figcaption>drawing credit goes to <a href="https://reddit.com/u/Sinyuri">u/Sinyuri</a></figcaption>
</figure>
<p>Even after learning to adapt to this, I still thought that it would be nice to be able to detect distinct groups among the twitter accounts that I followed. The impetus to finally start a project about this came when I started using cluster analysis algorithms in my machine learning class - the algorithms used seemed to be exactly the right idea for this kind of community detection. With that I set off on the task to collect and analyze the data from my own Twitter follow list, with clusters!</p>
<p>The work I've done since then is still in progress (mostly because the results I'm getting aren't that great yet), and as I make more progress I'll be making more posts about it!</p>
<p>All the code is available <a href="https://github.com/emsal1863/twitter_clustering_py">on Github</a>.</p>
<p>More details below!</p>
<!--more-->
<h2 id="the-simplest-aim-data-representation-of-a-twitter-account">The simplest aim: data representation of a Twitter account</h2>
<p>To set up the clustering, there has to be a way to represent each twitter account in my follow list in some form that can be analyzed. For this project, I decided to identify a Twitter account purely by the other accounts that they follow. I represent this in a vector representation that is just like a row of an adjacency matrix for a graph. This graph has a &quot;central&quot; account, and we build up the graph based on who that central account follows -- these are what we'll call the <em>accounts of interest</em>. These accounts of interest each follow other accounts (including some of each other), and we're interested in these accounts, too — we'll call these the <em>accounts to examine</em>.</p>
<div class="figure">
<img src="https://i.imgur.com/3r2YdWW.png" />

</div>
<div class="figure">
<img src="https://i.imgur.com/pQTQKgT.png" />

</div>
<p>If we take all the accounts that all our accounts of interest follow and deduplicate them, we have a set of nodes of size <span class="math inline">\(m\)</span>. If we take following as a (directional) edge between two nodes, we can represent a twitter account as a vector, the size of it being <span class="math inline">\(m\)</span> by <span class="math inline">\(1\)</span>. If we have <span class="math inline">\(n\)</span> accounts of interest, then our data is represented in the form of an <span class="math inline">\(n \times m\)</span> matrix. If account of interest <span class="math inline">\(i\)</span> follows account to examine <span class="math inline">\(j\)</span>, the cell in the <span class="math inline">\(i\)</span>th row and the <span class="math inline">\(j\)</span>th column will be a <span class="math inline">\(1\)</span>, otherwise it'll be <span class="math inline">\(0\)</span>. It's worth noting that, in almost all cases, <span class="math inline">\(m\)</span> will be much larger than <span class="math inline">\(n\)</span>, which is a serious, almost existential problem to this project that I will go into more detail about later in the post.</p>
<div class="figure">
<img src="https://i.imgur.com/GNge932.png" width="900" />

</div>
<h2 id="collecting-the-data">Collecting the data</h2>
<p>Now that I've decided on a representation for every twitter account in my follow list, I need to collect the data necessary to fill up the matrix with all of the edges present in our follow graph. The way that I originally was thinking of doing it was via the Twitter API, which would probably have been the most straightforward way to do it. However, Twitter heavily rate limits doing this: if you want to get a user's follow list, then you can only do so <a href="https://developer.twitter.com/en/docs/basics/rate-limits.html">15 times every 15 minutes</a> (look at the <code>GET friends/list</code> row), which is pretty unwieldy to work with.</p>
<p>To work around this, I wrote a selenium script in python that would go through a user's follow list just by going to the <code>/{user}/following</code> endpoint and collect the names of the accounts that they follow just from the HTML. Here's the relevant function:</p>

{{< highlight python >}}
def get_follow_list(username, your_username, password):
    browser = webdriver.Firefox()

    browser.get("https://twitter.com/{username}/following".format(**{'username' : username}))

    username_field = browser.find_element_by_class_name("js-username-field")
    password_field = browser.find_element_by_class_name("js-password-field")

    username_field.send_keys(your_username)
    password_field.send_keys(password)

    ui.WebDriverWait(browser, 5)
    password_field.send_keys(Keys.RETURN)

    try:
        ui.WebDriverWait(browser, 10).until(EC.presence_of_element_located((By.CLASS_NAME, "ProfileCard")))
    except:
        return []


    last_height = browser.execute_script("return document.body.scrollHeight;")
    num_cards = len(browser.find_elements_by_class_name("ProfileCard"))
    while True:
        browser.execute_script("window.scrollTo(0, document.body.scrollHeight)");

        try:
            new_height = ui.WebDriverWait(browser, 1.5).until(new_height_reached(last_height))
            last_height = new_height
        except:
            # print(sys.exc_info())
            break

    print("Loop exited; follow list extracted")

    ret = browser.execute_script("return document.documentElement.innerHTML;")
    browser.close()
    soup = BeautifulSoup(ret, 'html.parser')
    cards = soup.findAll("div", {"class": "ProfileCard"})
    screennames = [card.get("data-screen-name") for card in cards]

    return screennames
{{< / highlight >}}
<p>And here's a video of it in action:</p>
<iframe width="560" height="315" src="https://www.youtube.com/embed/K5r3ZQRzd3M" frameborder="0" gesture="media" allowfullscreen>
</iframe>
<p>In the end, I get a load of text files, each pertaining to an account's follow list, and the information in those text files is just simply the names of all the accounts they follow. Having that all stuffed in a folder, I set out processing the data.</p>
<h2 id="from-a-profile-to-a-vector">From a profile to a vector</h2>
<p>I did all of the clustering in Julia, not Python, because it's something that I've grown familiar with since starting my school's machine learning class and have started really admiring because of its surprising ease of use at its performance point. I encourage everyone with even a passing interest in scientific computing to try and learn it, it really has been fun so far.</p>
<p>We eventually need to get to the matrix representation of our data that I was describing earlier. I did this very messily and without much consideration for performance beyond basic common sense, so bear with me.</p>
<p>The following code reads all the followed accounts from the files and loads them into a dictionary. The keys of this dictionary are the followed accounts, and the values are all the accounts of interest that follow them.</p>

```julia
followed = Dict()

mutable struct FollowedAccount
    numFollowing::Int
    following::Array{String}
end

for (n, file) in enumerate(validFiles)
    strm = open(file, "r")
    followedAccs = readlines(strm)
    for acc in followedAccs
        if haskey(followed, acc)
            followed[acc].numFollowing += 1
            push!(followed[acc].following, usernames[n])
        else
            followed[acc] = FollowedAccount(1, [usernames[n]])
        end
    end
    close(strm)
end
```

<p>So, in <code>followed</code>, we now have a dictionary where we can find all the users following a particular account. We construct the data matrix by first initializing an <span class="math inline">\(n \times m\)</span> matrix of <span class="math inline">\(0\)</span>s and filling the cell in the <span class="math inline">\(i\)</span>th row and the <span class="math inline">\(j\)</span>th column with a <span class="math inline">\(1\)</span> if account of interest <span class="math inline">\(i\)</span> follows account-to-examine <span class="math inline">\(j\)</span>.</p>
<!-- HTML generated using hilite.me -->
{{< highlight julia >}}
(dim,) = size(followingToCheck)

(n,) = size(usernamesFiltered)
dataPoints = zeros(n,dim)

for (usernameIdx, username) in enumerate(usernamesFiltered)

    for (followedIdx, followedAcc) in enumerate(followingToCheck)
        if in(username, followed[followedAcc].following)
            dataPoints[usernameIdx, followedIdx] = 1
        end
    end
end
{{< / highlight >}}

<p>(Note: some variable names have been changed for easier reading in this post)</p>
<h2 id="what-the-cluster">What the cluster?</h2>
<p>Clustering algorithms are a form of unsupervised learning, meaning that there's no single &quot;right answer&quot; that they need to try to guess at. Clustering algorithms take data in the form of a bunch of points and partitions them into some distinct groups. In general (in all clustering algorithms I've seen so far), &quot;closer&quot; points are commonly grouped together, while points that are further apart tend to go in different clusters.</p>
<p>The notion of &quot;closer&quot; is obviously centrally important to clustering algorithms. This is why it was important for us to represent the things we want to cluster, Twitter accounts, as vectors: we can actually measure a quantifiable between any two of them by taking the norm of their difference.</p>
<p>The clustering algorithm used for this project is my own artisanally handcrafted implementation of <a href="https://en.wikipedia.org/wiki/DBSCAN">DBSCAN</a>, which I learned about in class. DBSCAN is a density-based clustering algorithm, meaning that it finds regions where many points exist in a small radius. The minimum number of points that it searches for, and what radius it searches within, are the two parameters of a DBSCAN model, <code>MINPTS</code> and <code>RADIUS</code>.</p>
<p>The algorithm works in pseudocode as follows:</p>
<pre><code>DBSCAN(MINPTS, RADIUS):
    For every point in the dataset
        If the point is already labelled as being part of the cluster, skip it
        Otherwise:
            If MINPTS points exist within RADIUS of the point:
                ExpandCluster around that point

ExpandCluster(p):
    Label p as belonging to a new cluster
    For every unlabelled point within RADIUS of p:
        call ExpandCluster around those points</code></pre>
<p>Here's the actual implementation:</p>

```julia
function densityCluster(dataPoints)
    n = size(dataPoints, 1)

    MINPTS = 5
    RADIUS = sqrt(n * 3 / 10)

    y = zeros(n)

    currentClusterNum = 1

    for i in 1:n
        if y[i] != 0
            continue
        end

        distances=[]
        for j in 1:n
            if i !== j
                push!(distances, norm(dataPoints[j, :] - dataPoints[i, :]))
            end
        end

        if sort(distances)[MINPTS] <= RADIUS
            println("calling expandCluster on point ", i)
            currentClusterNum = expandCluster!(y, dataPoints, i, currentClusterNum, RADIUS, MINPTS)
        end
    end

    return y
end

function expandCluster!(y, dataPoints, idx, currentClusterNum, radius, minPts)

    queue = [idx]

    visited = Set()

    n = size(dataPoints, 1)

    count = 0

    for o in queue
        if y[o] == 0
            y[o] = currentClusterNum
            count += 1
        else
            continue
        end

        (sortedPoints, distances, perm) = closestPoints(dataPoints, idx, n-1)

        for j in 1:n
            if distances[j] <= radius && y[perm[j]] == 0
                push!(queue, perm[j])
                push!(visited, perm[j])
            end
        end
    end

    if count < minPts
        for pt in visited
            y[pt] = 0
        end
        return currentClusterNum
    end

    return currentClusterNum + 1
end
```

<p>Of central importance to the <code>DBSCAN</code> algorithm is the notion of getting the closest points to a particular point, in order. It takes <span class="math inline">\(O(nd)\)</span> time to find these closest points (in my implementation I just call <code>sort</code> instead of just keeping track of the smallest points so I believe mine might actually be <span class="math inline">\(O(dn\log n)\)</span>). There's definitely been some fantastic work in data structures and algorithms that use approximations or other clever workings to make finding closest points faster, which is vital when your dataset is huge (which is the case for things like the datasets at large software companies). In any case, here's my implementation of <code>closestPoints</code>, which may cause you to cringe:</p>
<!-- HTML generated using hilite.me -->

```julia
function closestPoints(X::Array{Float64, 2}, i, numPoints)

    n = size(X, 1)

    distances=[]
    for j in 1:n
        push!(distances, norm(dataPoints[j, :] - dataPoints[i, :]))
    end

    return X[sortperm(distances)[1:numPoints], :], distances, sortperm(distances)
end
```

<h2 id="the-curse-of-dimensionality">The Curse of Dimensionality</h2>
<p>The DBSCAN algorithm is great, but it's sensitive to the choices of the two parameters <code>MINPTS</code> and <code>RADIUS</code>. I'm actually still trying to select the best choices for those two parameters, which can be difficult since it's hard to judge the output of the algorithm. Furthermore, like any data analysis algorithm, it won't produce any meaningful data if the data is unusable for whatever reason.</p>
<p>In this case, if I just ran DBSCAN on the matrix form of the data I collected from my Twitter follow list, it would <strong>be unlikely to produce any useful output</strong>. This is because of a problem known as <a href="https://en.wikipedia.org/wiki/Curse_of_dimensionality">the Curse of Dimensionality</a>. DBSCAN is a spatial clustering algorithm, which depends on dense clusters existing in the space you're exploring. Since the space we're exploring has a number of dimensions (<span class="math inline">\(m\)</span>) that is much, much higher than the number of data points (<span class="math inline">\(n\)</span>), the points are in general too distant to find any sort of dense regions at all.</p>
<p>To deal with this, I have to reduce the dimensions (the number of columns) of the matrix. The algorithm that I used for this is called <a href="https://en.wikipedia.org/wiki/Principal_component_analysis">principal component analysis</a>, or PCA. It does what the name suggests, figuring out the most principal components, the &quot;most important&quot; features, and reducing the dimensionality by transforming the matrix so that only the columns corresponding to those principal components are left.</p>
<p>I very naively use this algorithm without fully understanding details of how it works (I'll gain some understanding soon, though, and maybe write a post about it!), inputting a target dimensionality I want to achieve and just throwing PCA at my function. The specific library I used was <a href="https://github.com/JuliaStats/MultivariateStats.jl">MultivariateStats.jl</a>.</p>
<p>I have about 300 accounts of interest, so I reduce the dimensionality of the matrix from some number much greater than 300 to something decently below 300, like 40. I also experimented with standardizing each of the data points so that they're generally closer together.</p>

```julia
function rescale(A, dim::Integer=1)
    res = A .- mean(A, dim)
    res ./= map!(x -> x > 0.0 ? x : 1.0, std(A, dim))
    return res
end

reshapedData = convert(Matrix{Float64}, dataPoints)
X = rescale(reshapedData, 1)
M = fit(PCA, X'; maxoutdim = reduce_dim)
```

<p>So now, instead of a <span class="math inline">\(n \times m\)</span> matrix I now have an <span class="math inline">\(n \times 40\)</span> matrix, and since my <span class="math inline">\(n\)</span> is significantly greater than 40, I can do the DBSCAN on it without running into the Curse. I'm not totally sure if doing clustering on the output of PCA is entirely appropriate but with this particular set of parameters, I'm able to gain results that verge on the passable.</p>
<h2 id="struggles-and-things-i-tried-that-didnt-make-the-cut">Struggles, and things I tried that didn't make the cut</h2>
<p>The most difficult part of this experiment was being able to code a correct <code>expandCluster</code> function. At first, my algorithm was placing almost every data point in a single cluster. After much inspection of my code (with a regrettable lack of unit testing which I aim to fix sooner or later), I noticed that the algorithm wasn't handling the case where there were reachable points, but they were all already assigned to a cluster. I definitely had to rewrite the basic iteration through the queue a bunch of times.</p>
<p>Also, I didn't come up with this particular set of algorithms and representations on the first try. There were many things I tried before going ahead with them, some of which I may bring back when I go further into this.</p>
<p>At the outset, instead of using PCA, I used a much more naive metric for reducing dimensionality: eliminate accounts that weren't followed by at least, say, 15 people from my accounts of interest. I realized this wasn't a very smart approach when there may be a cluster of 14 people who each have in common that they follow an account that this threshold-based method would eliminate.</p>
<p>I also tried to use <a href="https://en.wikipedia.org/wiki/T-distributed_stochastic_neighbor_embedding">t-SNE</a>, but ended up opting for PCA because it seemed like there was even less of a hope of me being able to understand it and tweaking it accordingly in a reasonable timeframe. Perhaps one day.</p>
<h2 id="so-howd-it-do">So, how'd it do?</h2>
<p>The short answer is &quot;Interestingly.&quot; After I coded up the solution, I spent about a week (on and off, of course) just trying out different values for the parameters:</p>
<ul>
<li>The dimensionality reduction I want</li>
<li>The values for <code>MINPTS</code> and <code>RADIUS</code> for the DBSCAN algorithm</li>
</ul>
<p>Eventually, I settled on values of <span class="math inline">\(40\)</span> for the number of dimensions I wanted from PCA, <code>MINPTS = 5</code>, and <code>RADIUS = sqrt(n * 3 / 10)</code>. It's not because there was any sort of metric by which these are optimized against — the &quot;fitness&quot; of the output of a clustering algorithm is entirely subjective, after all. It's just because I found that that these choices gave 6 clusters (and the &quot;no-cluster&quot; assignment) for 300 data points that looked subjectively passable.</p>
<p>By passable, unfortunately, I don't mean that I can always find out what the clusters actually mean. I found that, perhaps, the distinct groups in my follow list that I originally thought were pretty distinguishable may have more unclear borders after all. For example, if I were to subjectively tag certain accounts as &quot;uni CS students&quot;, the algorithm found that the uni CS students were spread across the clusters it created.</p>
<p>Overall, although the clusters are pretty nonsensical, at least I was able to get from the algorithm putting everything in a single cluster to creating six distinct clusters.</p>
<p>Here are the six clusters. Note, there are actually only six clusters: &quot;Cluster 0&quot; consists of all the data points that can't be clustered together, i.e. &quot;outliers&quot; in some sense.</p>
<pre><code>Cluster 0: String[&quot;3ncr1pt3d&quot;, &quot;ACLU&quot;, &quot;alexcruise&quot;, &quot;allwantoo&quot;, &quot;AOCrows&quot;, &quot;biiigfoot&quot;, &quot;BillSimmons&quot;, &quot;brucesharpe&quot;, &quot;casskhaw&quot;, &quot;daviottenheimer&quot;, &quot;desi_inda_house&quot;, &quot;dom2d&quot;, &quot;dragosr&quot;, &quot;dylancuthbert&quot;, &quot;EmilyDreyfuss&quot;, &quot;EmilyGorcenski&quot;, &quot;Eric_Hennenfent&quot;, &quot;fl3uryz&quot;, &quot;FreedomeVPN&quot;, &quot;FunnyAsianDude&quot;, &quot;generalelectric&quot;, &quot;GonzoHacker&quot;, &quot;hacks4pancakes&quot;, &quot;HertzDevil&quot;, &quot;hiromu1996&quot;, &quot;hirosemaryhello&quot;, &quot;InfoSecSherpa&quot;, &quot;ivy_hollivana&quot;, &quot;JJRedick&quot;, &quot;josephfcox&quot;, &quot;JossFong&quot;, &quot;jstnkndy&quot;, &quot;katgleason&quot;, &quot;kelleyrobinson&quot;, &quot;KenRoth&quot;, &quot;KimDotcom&quot;, &quot;KirkWJohnson&quot;, &quot;larsvedo&quot;, &quot;ltkilroy&quot;, &quot;marktmaclean&quot;, &quot;MichaelSSmithII&quot;, &quot;MITEngineering&quot;, &quot;MITSloan&quot;, &quot;mzbat&quot;,&quot;natazilla&quot;, &quot;netflix&quot;, &quot;NGhoussoub&quot;, &quot;NimkoAli&quot;, &quot;nrrrdcore&quot;, &quot;Nullbyte0x80&quot;, &quot;ParameterHacker&quot;, &quot;Patrick_McHale&quot;, &quot;Pogue&quot;, &quot;Rachel__Nichols&quot;, &quot;rcallimachi&quot;, &quot;robbyoconnor&quot;, &quot;russwest44&quot;, &quot;SadiqKhan&quot;, &quot;screenjunkies&quot;, &quot;ShamsCharania&quot;, &quot;SHAQ&quot;, &quot;shutupmikeginn&quot;, &quot;skayyali1&quot;, &quot;sovietvisuals&quot;, &quot;swannodette&quot;, &quot;thejoshlamb&quot;, &quot;tinyrobots&quot;, &quot;tpolecat&quot;, &quot;t_sloughter&quot;, &quot;verainstitute&quot;, &quot;viiolaceus&quot;, &quot;wojespn&quot;, &quot;ZachLowe_NBA&quot;]

Cluster 1: String[&quot;acciojackie&quot;, &quot;actualrecruiter&quot;, &quot;adultswim&quot;, &quot;aleph7&quot;, &quot;antirez&quot;, &quot;__apf__&quot;, &quot;aprilmpls&quot;, &quot;ARetVet&quot;, &quot;Arjyparjy&quot;, &quot;avasdemon&quot;, &quot;avery_katko&quot;, &quot;AvidHacker&quot;, &quot;AvVelocity&quot;, &quot;azuresun&quot;, &quot;b0rk&quot;, &quot;baesedandy&quot;, &quot;BBCBreaking&quot;, &quot;beriwanravandi&quot;, &quot;blakegriffin32&quot;, &quot;Bourdain&quot;, &quot;briankrebs&quot;, &quot;campuscodi&quot;, &quot;chriszhu12&quot;, &quot;cjbecktech&quot;, &quot;Colossal&quot;, &quot;coolbho3k&quot;, &quot;cryptonym0&quot;, &quot;CryptoVillage&quot;, &quot;CybertronVGC&quot;, &quot;da_667&quot;, &quot;dennisl_music&quot;, &quot;dysinger&quot;, &quot;elpritchos&quot;, &quot;EtikaWNetwork&quot;, &quot;fggosselin&quot;, &quot;gommatt&quot;, &quot;GoogleDoodles&quot;, &quot;grisuy&quot;, &quot;HamillHimself&quot;, &quot;hannibalburess&quot;, &quot;HariniLabs&quot;, &quot;hemalchevli&quot;, &quot;hrw&quot;, &quot;ijlaal_&quot;, &quot;irfansharifm&quot;, &quot;JennyENicholson&quot;, &quot;kelhutch17&quot;, &quot;kevinmitnick&quot;, &quot;kporzee&quot;, &quot;Levus28&quot;, &quot;MalwareTechBlog&quot;, &quot;matthias_kaiser&quot;, &quot;MichelleGhsoub&quot;, &quot;MISSINGinCANADA&quot;, &quot;MITEECS&quot;, &quot;MonotoneTim&quot;, &quot;oviosu&quot;, &quot;pixpilgames&quot;, &quot;pud&quot;, &quot;radareorg&quot;, &quot;rendevouz09&quot;, &quot;rice_fry&quot;, &quot;rocallahan&quot;, &quot;saemg&quot;, &quot;samhocevar&quot;, &quot;sarahkendzior&quot;, &quot;Schwarzenegger&quot;, &quot;scottlynch78&quot;, &quot;sean3z&quot;, &quot;shadowproofcom&quot;, &quot;SylviaEarle&quot;, &quot;theQuietus&quot;, &quot;the_suzerain&quot;, &quot;Toby4WARD&quot;, &quot;toriena&quot;, &quot;trevorettenboro&quot;, &quot;ubcprez&quot;, &quot;UncleSego&quot;, &quot;unkyoka&quot;, &quot;user935&quot;, &quot;x0rz&quot;, &quot;xoreaxeaxeax&quot;]

Cluster 2: String[&quot;benheck&quot;, &quot;melissa05092&quot;, &quot;pwnallthethings&quot;, &quot;rishj09&quot;, &quot;uhlane&quot;, &quot;vgdunkey&quot;]

Cluster 3: String[&quot;BerniceKing&quot;, &quot;BetaList&quot;, &quot;biancasubion&quot;, &quot;BrandSanderson&quot;, &quot;FastForwardLabs&quot;, &quot;Jinichuu&quot;, &quot;theTunnelBear&quot;]

Cluster 4: String[&quot;brainpicker&quot;, &quot;byrook1e&quot;, &quot;CanEmbUSA&quot;, &quot;carolynporco&quot;, &quot;CatchaSCRapper&quot;, &quot;charmaine_klee&quot;, &quot;charmwitch&quot;, &quot;christenrhule&quot;, &quot;CockroachDB&quot;, &quot;codeahoy&quot;, &quot;colour&quot;, &quot;CommitStrip&quot;, &quot;compArch1&quot;, &quot;coolgaltw&quot;, &quot;countchrisdo&quot;, &quot;cperciva&quot;, &quot;cursedimages_2&quot;, &quot;cursedimages&quot;, &quot;DenisTrailin&quot;, &quot;dennis_lysenko&quot;, &quot;dronesec&quot;, &quot;Ean_Dream&quot;, &quot;elnathan_john&quot;, &quot;encryptme&quot;, &quot;e_sjule&quot;, &quot;faceb00kpages&quot;, &quot;Fox0x01&quot;, &quot;FreedomofPress&quot;, &quot;jessysaurusrex&quot;, &quot;KamalaHarris&quot;, &quot;mononcqc&quot;, &quot;norm&quot;, &quot;racholau&quot;, &quot;reiver&quot;, &quot;SWUBC&quot;, &quot;Wugabuga_&quot;, &quot;YashTag1&quot;]

Cluster 5: String[&quot;GEResearch&quot;, &quot;haikus_are_easy&quot;, &quot;hasherezade&quot;, &quot;i0n1c&quot;, &quot;i_cannot_read&quot;, &quot;ididmorepushups&quot;, &quot;idiot_joke_teen&quot;, &quot;INFILTRATION85&quot;, &quot;internetofshit&quot;, &quot;InthelifeofLuke&quot;, &quot;InuaEllams&quot;, &quot;jacobtwlee&quot;, &quot;japesinator&quot;, &quot;jeriellsworth&quot;, &quot;Jiyanxa&quot;, &quot;JoelEmbiid&quot;, &quot;JohnBoyega&quot;, &quot;Junk_lch&quot;, &quot;justinschuh&quot;, &quot;jxson&quot;, &quot;kelseyhightower&quot;, &quot;KingJames&quot;, &quot;kinucakes&quot;, &quot;klei&quot;, &quot;kmuehmel&quot;, &quot;Kosan_Takeuchi&quot;, &quot;krypt3ia&quot;, &quot;ksenish&quot;, &quot;KurtBusiek&quot;, &quot;l0stkn0wledge&quot;, &quot;Legal_Briefs08&quot;, &quot;LibyaLiberty&quot;, &quot;MachinePix&quot;, &quot;malwareunicorn&quot;, &quot;motomaratai&quot;, &quot;nwHacks&quot;, &quot;rei_colina&quot;, &quot;SciBugs&quot;, &quot;skamille&quot;, &quot;susanthesquark&quot;, &quot;trufae&quot;]

Cluster 6: String[&quot;Mechazawa&quot;, &quot;MiraiAttacks&quot;, &quot;MoebiusArt1&quot;, &quot;msftmmpc&quot;, &quot;musalbas&quot;, &quot;naval&quot;, &quot;ndneighbor&quot;, &quot;nigguyen&quot;, &quot;omniboi&quot;, &quot;oocanime&quot;, &quot;PanguTeam&quot;, &quot;PengestMunch&quot;, &quot;ProfSaraSeager&quot;, &quot;progpaintings&quot;, &quot;prozdkp&quot;, &quot;ramonashelburne&quot;, &quot;RascherMichael&quot;, &quot;readingburgers&quot;, &quot;regretsuko&quot;, &quot;repjohnlewis&quot;, &quot;riibrego&quot;, &quot;RoguePOTUSStaff&quot;, &quot;rowletbot&quot;, &quot;Rutiggers&quot;, &quot;s0lst1c3&quot;, &quot;s7ephen&quot;, &quot;sadserver&quot;, &quot;seanswush&quot;, &quot;SenJohnMcCain&quot;, &quot;sheer_hope&quot;, &quot;SheerPriya&quot;, &quot;shiunken&quot;, &quot;stinkbug&quot;, &quot;swipathefox&quot;, &quot;Syphichan&quot;, &quot;tajfrancis&quot;, &quot;tea_flea&quot;, &quot;ternus&quot;, &quot;thealexhoar&quot;, &quot;thebenheckshow&quot;, &quot;theHeckwKaren&quot;, &quot;TheRujiK&quot;, &quot;ThisIsJBird&quot;, &quot;tokyomachine&quot;, &quot;torbooks&quot;, &quot;Tyler_J_Mallloy&quot;, &quot;UBCAvocadoWatch&quot;, &quot;VessOnSecurity&quot;, &quot;VGArtAndTidbits&quot;, &quot;weworkremotely&quot;, &quot;WJ_VJ&quot;, &quot;worrydream&quot;, &quot;YO_SU_RA&quot;]</code></pre>
<h2 id="whats-next">What's next?</h2>
<p>This is really still a work in progress so I wouldn't even consider this project to even be complete; the purpose of this post is more to share my first explorations into this space, as a relative novice to data science. The next steps for this project would probably involve a lot of research and a lot more implementation and parameter tweaking. Some specific areas are:</p>
<ul>
<li>Understanding what PCA actually does and tweaking the parameter of the output dimension in a more informed way
<ul>
<li>Toggling rescaling</li>
</ul></li>
<li>Understanding the <em>implications</em> of using a PCA result as the input to a clustering algorithm</li>
<li>Using different clustering algorithms on the data, like <a href="https://en.wikipedia.org/wiki/K-means_clustering">k-means</a></li>
<li>Possibly automating parameter tuning (in a brute-force way, rather than an optimization way because any optimization is still unclear)</li>
<li>Trying to use more features based on tweet, biography, and other forms of content in addition to the follow graph data</li>
<li>Try to use different algorithms for dimensionality reduction, like t-SNE or auto encoders</li>
</ul>
<p>I'm going to take a bit of a break from this project for now to focus on other projects, but look forward to an eventual follow-up with my further experimentation in coming up with a better algorithm!</p>
