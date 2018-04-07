---
layout: post
title: Project McNulty
---

## The Wonderful World of Cable News Commentary

For this project, I dove into nonpartisan, unbiased, boring world of primetime cable news shows (who am I kidding, that wouldn't be any fun). I chose two hosts from CNN, Fox News, and MSNBC (the so-called "Big 3" Networks). These networks allowed me to access and compare sources across the political spectrum. The image below shows news sources of all types. I've summarized each choice below the image.

<img src='//stokvis4.github.io/images/News_Sources.jpg'>

* CNN
	* Description - Despite Trump's continual assualt on CNN for their liberal bias, CNN has embraced the sensationalism wholeheartedly. 
	* Hosts:

		* 7pm slot - Erin Burnett (Erin Burnett OutFront)
		* 8 & 9pm slot - Anderson Cooper (Anderson Cooper 360)

* Fox News
	* Description - Fox News has been the leader in cable news for 194 months for a reason. It embraces its contrarian views and combines its sensationalist news with a strong conservative bias, separating itself from the rest of cable news.
	* Hosts:

		* 8pm - Tucker Carlson (Tucker Carlson Tonight)
		* 9pm - Sean Hannity (The Hannity Show)

* MSNBC
	* Description - Although not as sensationalist as Fox News, MSNBC certainly fills the liberal side of the spectrum. It has successfully filled the role of opposition during the Trump administration similar to how Fox News did during the Obama administration.
	* Hosts:

		* 8pm - Chris Hayes (All In with Chris Hayes)
		* 9pm - Rachel Maddow (The Rachel Maddow Show)

### Objective and Overview

This project was focused on unsupervised learning and as my project developed, I started noticing distinct differences between Fox News hosts and the other cable news networks. Considering Fox News is the most watched network of the three (I personally watch it sometimes because I find the way they frame issues fascinating), I wanted to discover what the primetime hosts were doing that made them so successful. I combined these findings with behavioral economic principles in order to further explore their success.

### Sources

For CNN and MSNBC, I was able to scrape their respective sites for the transcipts from each host I wanted. Fox News only had Sean Hannity's transcripts available so I used Archive.org to scrape Tucker Carlson.

### Scraping

I am going to use MSNBC as an example for how I scraped transcripts. 

The first step is to scrape the urls for every transcript. I used Selenium although other solutions including Scrapy could be used. I stored this in a list called "maddow_urls"
```python

driver.get('http://www.msnbc.com/transcripts/rachel-maddow-show/2016/2')

maddow_urls = []
for year in [2016,2017,2018]:
    for month in range(1,13):
        if year ==2018 and month ==3:
            break
        else:
            driver.get('http://www.msnbc.com/transcripts/rachel-maddow-show/' + str(year) + '/' + str(month))
            a = 0
            while True:
                a += 1
                selector_rm = '//*[@id="block-system-main"]//div[@class="item-list"][1]//li[' + str(a) + ']//a'
                try:
                    url = driver.find_element_by_xpath(selector_rm).get_attribute('href')
                    maddow_urls.append(url)
                except:
                    break

maddow_urls

```

Once I have the URLs, I wanted to extract specific information and metadata about each transcript. For each transcript, I get the Date (from the URL), URL, and the content.

```python

date_rm = []
url_rm = []
content_rm = []
for maddow in maddow_urls2:
    driver.get(maddow)
    date = maddow[52:62]
    date_rm.append(date)
    url = maddow
    url_rm.append(url)
    c = driver.find_element_by_xpath('//*[@id="block-system-main"]//div[@class="panel-pane pane-entity-field pane-node-body"]//div[@class="pane-content"]').text
    content_rm.append(c)


```


The next few steps involve organizing the data, removing the copyright information contained at the bottom of each transcript and adding a few more data points.

```python

def remove_msnbc_copyright(x):
    x = x[:x.find('THIS IS A RUSH TRANSCRIPT.')]
    return x
    
maddow_all = [date_rm, url_rm, content_rm]
df_rm = pd.DataFrame(maddow_all)
df_rm = df_rm.transpose()
df_rm.columns = ['Date','Content','URL']
df_rm['Title'] = 'Rachel Maddow'
df_rm['Show'] = 'The Rachel Maddow Show'

df_rm.rename(columns={'Content':'URLs','URL':'Content'}, inplace=True)


df_rm['Content'] = df_rm['Content'].apply(remove_msnbc_copyright)

```


##### DataFrame Preview
<img src='//stokvis4.github.io/images/Maddow_head.png'>


### Cleaning Process

This process is repeated for each host. 

Once I've scraped every host's transcripts, I format the date and add the networks.

```python
df_ch['Date'] = pd.to_datetime(df_ch.Date, format='%Y-%m-%d')
df_rm['Date'] = pd.to_datetime(df_rm.Date, format='%Y-%m-%d')
df_ebo['Date'] = pd.to_datetime(df_ebo.Date, format='%y/%m/%d')
df_ac['Date'] = pd.to_datetime(df_ac.Date, format='%y/%m/%d')
df_h['Date'] = pd.to_datetime(df_h.Date, format = '%Y/%m/%d')
df_tc['Date'] = pd.to_datetime(df_tc['Date'], format="%B %d, %Y")
df_ac['Show'] = df_ac['Show'] + ' - ' +  df_ac['ShowNum']
df_ac.drop('ShowNum', axis=1, inplace=True)

df_ch['Network'] = 'MSNBC'
df_rm['Network'] = 'MSNBC'
df_ebo['Network'] = 'CNN'
df_ac['Network'] = 'CNN'
df_tc['Network'] = 'Fox News'
df_h['Network'] = 'Fox News'


```


Using Rachel Maddow as an example again, I created a function that removes new line breaks and splits the content every time someone speaks. The character I look for in every case is a "Maddow:". This splits the content into a list that starts with Maddow speaking every time. This, however, contains information about each following speaker until Maddow speaks again. As a result, I do a second split on ":" and include the first item only. After that, I look at the last character of each item and remove any information that matches any one of these characters "(),-", a space or any capitalized letter.

```
def MSNBC_only_host(line, host):
	present = host + ':'
	if present in line:
	    line = line.replace('\n', ' ')
	    if host == 'MADDOW':
	        splitter = '(RACHEL ' + host + ', MSNBC HOST\:|' + host + '\:)'
	    else:
	        splitter = '(Chris ' + host + ', MSNBC HOST\:|' + host + '\:)'
	        print (splitter)
	    line = re.split(splitter, line)
	    maddow_words = []
	    holder = []
	    for test in line[2::2]:
	        maddow_words.append(test.split(':')[0])
	    try:
	        for lines in maddow_words:
	            while (re.match('(\(|\)|\ |\,|\-|[A-Z])', lines[-1]) is not None):
	                lines = lines[:-1]
	            holder.append(lines)
	        holder = ' '.join(holder)
	        return holder
	    except:
	        return ' '
	else:
		return 'Guest'

df_rm['Commentator_Only'] = df_rm['Content'].apply(lambda line: MSNBC_only_host(line, 'MADDOW'))
df_ch['Commentator_Only'] = df_ch['Content'].apply(lambda line: MSNBC_only_host(line, 'HAYES'))

```

The process is previewed below on a small segment of the entire transcript from January 22, 2016:

Remove all '\n' characters. Those characters are highlighted below in yellow:

<img src='//stokvis4.github.io/images/Transcript_highlighting_newlines.jpg'>


Split the content into blocks using a regex statement. The following image contains the list that remains after the split:

<img src='//stokvis4.github.io/images/Transcript_in_list.jpg'>


The final product contains only the transcript of the host while removing any unwanted characters that remain:

<img src='//stokvis4.github.io/images/Final_Transcript.jpg'>


This process is repeated for every transcript from each host. The process is slightly more complicated for Tucker Carlson since it is pulled from Archive.org. In this specific case, advertisements and time markings are removed from the document. 

Additionally, for each transcript, if the host is on vacation, I denote the content as 'Guest'.


### Exploratory Data Analysis

#### Sentiment

##### General Sentiment

At this point, I've gotten to the cool part of the analysis. With my cleaned host-only transcripts, I can start performing sentiment analysis on them. I chose to use VADER Sentiment analyis (package can be found [here](https://github.com/cjhutto/vaderSentiment)). VADER is a powerful tool that allowed me to plot the positive and negative sentiment on each host.


<img src='//stokvis4.github.io/images/sentiment.png'>

In the visualization above, you can easily see the positive (on the x-axis) and negative (on the y-axis) sentiment of the hosts by network. Visual analysis easily shows that Fox News hosts embrace emotion, particularly negative emotion, in their broadcasts. Stealing from behavioral economics, this increased negative sentiment highlights the negativity bias. 

Wikipedia says Negativity Bias "refers to the notion that, even when of equal intensity, things of a more negative nature (e.g. unpleasant thoughts, emotions, or social interactions; harmful/traumatic events) have a greater effect on one's psychological state and processes than neutral or positive things." Negativity bias is a result of evolution and protects us from threat. Rick Hanson explains further in the following quote:


>The nervous system has been evolving for 600 million years, from ancient jellyfish to modern humans. Our ancestors had to make a critical decision many times a day: approach a reward or avoid a hazard — pursue a carrot or duck a stick.
>
>Both are important. Imagine being a hominid in Africa a million years ago, living in a small band. To pass on your genes, you’ve got to find food, have sex, and cooperate with others to help the band’s children (particularly yours) to have children of their own: these are big carrots in the Serengeti. Additionally, you’ve got to hide from predators, steer clear of Alpha males and females looking for trouble, and not let other hunter-gatherer bands kill you: these are significant sticks.
>
>But here’s the key difference between carrots and sticks. If you miss out on a carrot today, you’ll have a chance at more carrots tomorrow. But if you fail to avoid a stick today – WHAP! – no more carrots forever. Compared to carrots, sticks usually have more urgency and impact.

Source: [http://www.rickhanson.net/how-your-brain-makes-you-easily-intimidated/]

A more relevant example may be why the NRA is so successful at preventing the gun control. According to research, conservatives have larger amygdalae, a cluster of the brain believed to beassociated with basic pleasure and fear responses ([source](https://www.vox.com/2015/12/4/9845146/mass-shootings-gun-control)). This NRA ad shows how they utilize negative sentiment with loss aversion to tell their message.

<img src='//stokvis4.github.io/images/nra-obama.jpg'>


This, in combination with loss aversion bias, shows one of the reasons why the NRA is so successful at getting voters to turn out.

##### Sentiment over Time

I also used sentiment to understand 2017 politically and was able to map significant political events to divergences in sentiment between Rachel Maddow and Sean Hannity.


<img src='//stokvis4.github.io/images/timeline.png'>


For the above graph, I used a 5 day moving average in order to remove rapid positive and negative swings that were common for each host. 

###### May 2017

Here we see the first big divergence between Maddow and Hannity. On May 9, President Trump fired James Comey which was quickly followed by Trump's comments to the Russian ambassador which stated "I just fired the head of the FBI. I faced great pressure because of Russia. That's taken off." Shortly following that, Rod Rosenstein appoints special counsel Robert Mueller on May 19. 


If we take a quick look at the some of the intro content of May 9th, the day of James Comey's firing, we see very strong differences in the tone and sentiment of each host:

Hannity:

>But first, kicking Comey to the curb! This is the first step in President Trump draining the deep state swamp. And that is tonight's and perhaps my most important "Opening Monologue" ever.
>
>And welcome back to "Hannity."
>
>All right, we're going to cut through all of the noise, all of the nonsense on this program tonight and tell you exactly what the left and destroy Trump media will not tell you.
>
>James Comey, the former now FBI director, is a national embarrassment. It's that plain, it's that simple. And frankly, he's very lucky that President Trump kept him around this long because of his now unhinged and very erratic behavior.
>
>Now, firing James Comey was absolutely the single right thing for this president to do. It's good for the country, and let me tell you why.
>
>Comey has failed you, the American people, on a spectacular level! And at every single turn, the FBI director disrespected the Constitution, he showed he does not care about the equal application of the rule of law being applied equally American. He has now stood by while our 4th Amendment rights have been trampled upon. And worst of all, he has created in this country how a two-tiered justice system, one for Hillary and Bill Clinton, and one for the rest of America. It's become a travesty. Comey tonight should be ashamed of himself.  
>
>Now, let's start with facts, and Hillary Clinton and the private email server that she used purposely to circumvent what is known as congressional oversight. Here are the facts, plain and simple. Hillary Clinton's server contained top-secret special access programs -- in other words, the highest level of classified information on her computer. She deleted over 30,000 emails on that computer, claiming they were personal. They were about yoga, weddings, grandchildren and emailing Bill Clinton, who never had an email account. This was a lie from the get-go.
>
>And what's so despicable about this is back in July, when Comey made his big announcement on Hillary Clinton and the investigation, he acknowledged all of these facts, which, by the way, is an acknowledgment that crimes were committed!


Maddow:

>Thanks to you at home for joining us for the next hour.  What a day, huh? 
>
>You have been seeing live images of a plane sitting on a tarmac now moving down a taxiway in Los Angeles. This is a private plane of some kind. It's not unusual for federal agencies, especially big ones like the department of justice or the FBI to have private planes at their disposal. 
>
>Director Comey was in Los Angeles for, I think, what was supposed to be a recruiting event today, an event that was canceled. There was some logistical question once he was fired today by the White House as to what would physically happen to him in the immediate aftermath of his firing. He was removed effective immediately. 
>
>So, right now, he is no longer the director of the FBI.  What we believe that Director Comey is on that plane, which is now taxiing down the runway in Los Angeles, presumably they'll be flying back to the east coast, presumably, the FBI headquarters in Washington.  He'll be heading back to Washington. 
>
>But all of this is unscripted at this point.  All of this is unprecedented. There are historical parallels to what happened today, but there's never been anything like this before.  One instance previously in U.S. history in which an FBI director has been fired by as president. 
>
>That was a very different circumstance.  It was President Bill Clinton at the time.  The FBI director who was fired was William Sessions.  There were, in effect, abuse of office concerns that had been documented against him by the Department of Justice.  Things like using a Department of Justice aircraft to fly to see his family.  Things like using department resources to build a fence around his house that didn't seem to have any security purpose.  And maybe was just because he wanted a fence at this house. 
>
>Those kind of concerns that led to Sessions leaving.  William sessions being fired by president Clinton in the '90s.  That's the only precedent we've got for an FBI director being fired. 
>
>We're left as we watch these remarkable scenes and we wonder what's going to happen next here with James Comey – we're left to find other context, other analogies that make this make sense. 
>
>In 1972, there was a tape made in the Oval Office.  It was made on June 23rd, 1972.  In that tape, the then-President of the United States, Richard Nixon, and his chief of staff, H.R. Halderman, they talked about how they would cover up what they knew about the Watergate break-in, which the Nixon administration had orchestrated.  And there they were on tape talking about how to beat the investigation into it, how to cover it up. 
>
>And that tape was released to the public on August 5th, 1974.  The Supreme Court had ordered the president to release that tape and ultimately, he relented and released that tape. 
>
>By then, by the time that Watergate tape, that Oval Office tape was released, the Watergate scandal was quite ripe.  Impeachment proceedings were well under way already.  There were 11 Republicans on the House Judiciary Committee who voted against impeaching Richard Nixon. 
>
>But when that tape came out, that August, all of the 11 Republicans on that judiciary committee had said they would not impeach Nixon.  They all said hearing that tape, that they would change their votes.  That they would vote to impeach.


From a visual perspective, we can see distinct differences in tone. Sean Hannity immediately goes on the offensive against Comey while Maddow starts with historical precedence. If I apply, VADER sentiment to these commentaries, we see a stark difference in polarity. 

| Host        | Positive           | Negative  |
| ------------- |:-------------:| -----:|
| Hannity      | 0.055 | **0.088** |
| Maddow      | **0.087**      |   0.056 |


### Catchphrases

One of the most noticeable features of watching any Hannity or Carlson show is the prevalence of catchphrases. These memorable statements help Fox News hosts establish themselves clearly and allow them to position themselves in relation to other media sources. Using n-gram analysis, these catchphrases stand out compared to MSNBC and CNN hosts. 

For Hannity, it is "Destroy Trump Media" and for Carlson, it is "The Sworn Enemy of Lying, Pomposity, Smugness, and Group-Think" (analysis picks up variants of this statements). Both of these statements end up in the top 10 in n-grams. For comparison, Maddow's top 10 includes various media sources and gratitudes while Anderson Cooper's top 10 includes various media sources, Donald Trump and his son, and interview-oriented statements.

For comparative purposes, we can observe the effectiveness of Donald Trump's "Low Energy Jeb" and "Crooked Hillary" had on the election path. Each of these statements synthesized narratives that were short, witty, and, most importantly, memorable.

[![Trump Nicknames](http://static.businessinsider.com/image/571aa0a552bcd020008be706-750.jpg)](https://youtu.be/l6LN_--HjjE)


### Create the Enemy

The final area I looked at was Topic Modeling. Fox News hosts, especially Sean Hannity, have effectively created an enemy in the Clintons and use the idea of the Clintons to color many of the topics that were covered through the year. When I broke Sean Hannity's transcripts into 7 topics, Hillary Clinton comes up in 3 of the 7 topics. While she may be expected to be included in the Steele Dossier/Russian investigation, she is also mentioned in the FBI investigation, as well as mentioned in #MeToo conversation. These 2 topics are framed with her in mind. Take, for example, the November 15th episode when accusations against Roy Moore were coming out:


>Judge Roy Moore writes an open letter to me after last night I demanded answers. We will share that with you. Also, three more accusers down in Alabama. Now, my response is coming up later in the show.
>
>Plus, Democrats, liberals and the media, **they're finally now today having their day of reckoning when it comes to Bill Clinton and his wife Hillary, and the many allegations of the sexual misconduct, the mistreatment of women. Because for decades these accusations, they have dodged and been dismissed and downplayed by the left in this country, by the media in this country, and even worse. Those accusers were smeared, slander, besmirched, including by Hillary Clinton herself, and what is so inexcusable is that the Democrats and the media for 30 years did nothing to stop this from happening to protect women.**
>
>Instead, they were complicit. They enabled all of this to happen. Those women all suffered, and now they lecture everybody else. We will expose this blatant, despicable, disgusting hypocrisy in tonight's breaking news monologue.
>
>All right, with new allegations of sexual misconduct now swirling around US Senate candidate Judge Roy Moore, the left and the liberal media, they are finally starting to re-examine what is their deplorable defense over the years: Bill Clinton's long history of disturbing sexual abuse allegations.
>
>It is taken almost 30 years for the left, the liberal destroy Trump media to finally start admitting Bill Clinton targeted women for decades with criminal impunity. Take a look at this headline from a liberal publication -- a little late -- The Atlantic: Bill Clinton: A Reckoning. Feminists saved the 42nd president of the United States in the 1990s. They were on the wrong side of history, is it finally time to make things right?" Yes, it's beyond time. And then there's this op-ed from the New York Times and as the headline, quote, "I believe Juanita."

When Al Franken, John Conyers, and Matt Lauer got caught up in the MeToo movement, he mentions the Clintons again as a method to frame the issue:

>Then in 2008 when Franken was running for the Senate, he actually apologized for the comments we just showed you. But it turns out Franken was lying. How do we know? Because in his new book, Franken says he faked the apologies in order to win votes. He writes, "To say I was sorry for writing a joke was to sell out my career, to sell out who I've been my entire life, and I wasn't sorry. When I had written Porn-o-Rama or pitched that stupid Lesley Stahl joke at 2 in the morning, I was just doing my job." His apologies aren't real.
>
>Then there is the Clintons. Now, for those of you in the mainstream media, pay attention. Why is this important? Because for 30 years -- 30 -- Democrats on behalf of the Clintons have smeared, slandered, besmirched, victim shamed anyone and everyone who dared to tell what turned out to be the truth about Bill Clinton. And the media has always been their willing accomplices in and out this. They have been complicit up to and including through last year.
>
>The only reason now some in the political world are finally just starting to acknowledge what Clinton did is because Bill and Hillary Clinton are no longer important. So now that it's politically expedient, now that they can't hurt them, the Clintons that is, members of the media -- even some, but only some -- are only beginning to start to call out the deplorable behavior. So it's not about principle for them, it's about, well, from 1991 until 2016, the media defends the Clintons on everything regardless of how damaging or disgraceful the allegations were and, well, they can do it now. They can say maybe it wasn't right, but I believe Juanita.
>
>OK, well, Hillary Clinton continues to smear, slander, besmirched her husband's accuser today. Just take a listen to the latest attack by Hillary.

- October 20, 2017

These instances are not coincidental but done in order to effectively frame an issue. Stealing from Daniel Kahneman, "What you see is all there is". Hannity frames each of these issues in order to portray a narrative in a very effective manner.

Even when it comes to the Uranium One deal, Hannity places Clinton squarely in the center of the issue. He frames the narrative as focused around Clinton: 

>Now, in The Hill report, they explained that despite all of the evidence that they had of wrongdoing against Russia, the Obama Justice Department, for some unknown reason chose not to act and pursue charges. What did they do instead? The Obama administration then went on to approve business deals that benefited Russia, Putin, Moscow greatly while these threats to our national security were actually going on. These include the 2010 -- we've talked about in a lot, now we have more details -- that Uranium One deal. **Hillary Clinton, she signed off on it. She helped approve it.** That gave Vladimir Putin control, over 20 percent of America's uranium. Uranium is the foundational materials so you can build nuclear weapons. We'll have a lot more on that in just a second... Now we know there's bribery kickbacks, extortion, money laundering, I want to know where the money really came from, that went to the Clinton foundation. I want to know where that money came from. What is it funneled through Latvia or any of these groups? Who gave the money? **$145 million is a lot of money. Who benefited? It was Putin and Russia. That is the question.**

- October 17, 2017

These allegations prompted Shepherd Smith, a host from Hannity's own network, to confront the allegations and provide clarity into the actual happenings of the deal. 

{::options parse_block_html="false" /}

<div class="center">
<blockquote class="twitter-video" data-lang="en"><p lang="en" dir="ltr">Fox News reports on Uranium One.  Shepard Smith uses detailed facts &amp; graphs to explain why there is no reason to investigate.  It is a false controversy meant to draw our attention from real issues within the Trump administration.   <a href="https://t.co/Kx84vsTGDZ">pic.twitter.com/Kx84vsTGDZ</a></p>&mdash; ☇RiotWomenn☇ (@riotwomennn) <a href="https://twitter.com/riotwomennn/status/930594139959365632?ref_src=twsrc%5Etfw">November 15, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>


With all this combined, when compared to CNN and MSNBC hosts, Clinton was only mentioned in relation to the 2016 presidential campaign. 


<img src='//stokvis4.github.io/images/Topic_Modeling.png'>

When we look at the topics generated by Tucker Carlson and Sean Hannity together in comparison CNN and MSNBC, we really see differences in terms of content that is being talked about and how hard it is to reconcile political differences. For Fox News, the media, the Clintons, immigration, and the FBI are common topics while CNN and MSNBC are frequently talking about President Trump, the Special Counsel Investigation, and Trump Campaign. 

One area I wanted to focus one that both have in common is Russia. For Fox News, only 1 topic of the 7 mentions Russia. In comparison, 2 of the 7 for MSNBC and CNN cover Russia. You can see the groupings below:

**Fox News**
>uranium clinton hillary russian russia nuclear informant bribery john deal evidence solomon dnc podesta scandal

**CNN / MSNBC**
>trump meeting jr russian donald campaign know kushner russia mails manafort lawyer jared government information
>
>comey fbi president investigation director flynn memo house justice russia attorney mueller fired white trump

Although it can be considered a highly controversial subject, it is interesting to see how disparate these worlds are. 

In the end, all networks use framing as a tool, however, Fox News, in my belief, uses this to its full capacity. The continued demonization of the Clintons allow hosts like Hannity to associate topics with them in effect allowing them to direct the narrative in ways they view more favorable.  


### Conclusion

Cable news continues to play a dominant effect in the media. Whether this is productive or not for society, it is important to understand the delivery of such content. Fox News has been the leader for 194 months for a reason. The hosts embrace higher levels of negative sentiment than other networks, they craft witty catchphrases that their audiences can easily remember, and they effectively frame various issues using individuals that are despised by their audiences. 





