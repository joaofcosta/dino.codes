---
title: "How to make Twitter preview your website links"
date: 2020-01-07T16:19:41Z
draft: false
description: "Blogpost on how to update your website so that whenever you
tweet about it Twitter displays a preview of the content."
---

If you use Twitter you may have already seen a URL tweeted like this:

![Twitter Card Summary](/twitter_card_summary.png)

However, you've also probably noticed not every tweet with a URL appears that
way. That's because not all websites support [Twitter
Cards](https://developer.twitter.com/en/docs/tweets/optimize-with-cards/overview/abouts-cards)
.

In short, Twitter Cards let you display links more richly, by displaying a card
with more useful information than a simple URL. And the good news is, if you're
a website owner it's very easy to update your website so that Twitter can
display Twitter Cards for your website.

There are multiple types of cards you can use, namely:

- [Summary](https://developer.twitter.com/en/docs/tweets/optimize-with-cards/overview/summary)
- [Summary With Large Image](https://developer.twitter.com/en/docs/tweets/optimize-with-cards/overview/summary-card-with-large-image)
- [App](https://developer.twitter.com/en/docs/tweets/optimize-with-cards/overview/app-card)
- [Player](https://developer.twitter.com/en/docs/tweets/optimize-with-cards/overview/player-card)


In this blog post, I'm going to cover how to enable your website's links to
display summary cards but the knowledge should apply to all of the types of
cards, you'll just need to check the specific documentation for the type you're
using.

## Meta Header Tags

For Twitter to be able to display a URL as a card there needs to be
specific `meta` tags in the HTML header for that page. In our case, since we're
using the summary card, these are the meta tags we need to setup:

- `twitter:card` - Type of card to be used, we'll set this to `"summary"`
- `twitter:title` - The title to be displayed in the card.
- `twitter:site` - The Twitter @username the card should be attributed to. In
  this case, I used my website domain.
- `twitter:description` - A description, summarising the content of the URL.
- `twitter:image` - The URL for the image to be displayed on the card.

Note that only `twitter:card` and `twitter:title` are required for this to
work, but for the sake of completeness, and user-friendliness, I'm also
going to use the rest of the tags.

Here's how these tags look like in the final HTML file:

```html
<html>
    <head>
        <meta name="twitter:card" content="summary">
        <meta name="twitter:site" content="@joaofcosta">
        <meta name="twitter:image"
          content="https://i.imgur.com/ilEldyU.jpg">
        <meta name="twitter:title"
          content="Python Tuple Syntax Is Confusing">
        <meta name="twitter:description"
          content="Post about how Python's tuple syntax
            can make you run around trying to fix bugs
            that are easily avoided.">
        ...
    </head>
    <body>
        ...
    </body>
</html>
```

With the setup above, here's how your that page's URL will appear whenever
it is tweeted:

![Rendered Summary Card](/twitter_card_summary_legend.png)

Comparing that with tweeting only the URL you can see that it's much more
attention-grabbing and also allows the user to quickly infer what the URL is
about by using the description, it's a win-win!

If you wish for the image to be bigger you can also use the
`summary_large_image` value, instead of `summary`, for the `twitter:card`
meta tag. In that case, the URL would be displayed like this:

![Rendered Summary With Large Image
Card](/twitter_card_summary_with_large_image_legend.png)

The Twitter Card is now bigger and the image is now the main focus of
attention but the user is still able to capture information from the
description.

## Testing

If you decide to update your website's HTML so that Twitter can display these
types of cards you might be wondering how you can test that everything is set up
correctly.

Luckily, Twitter provides a
[Twitter's Card Validator](https://cards-dev.twitter.com/validator) where you can easily plug
the URL you'd be sharing and it'll render a preview of the card that would be
displayed, or not, were the URL to be shared.

## Conclusion

I think anyone that uses Twitter regularly can attest that it is
way better to see a Twitter Card for a website than a URL on a
Tweet.

As such, if you're the owner of a website, and usually share links to that same
website, I think it's a worthwhile investment to update your website to include
these meta tags, it's something really quick and easy to do that will probably
gather more visits to your website.

Finally, if you end up getting stuck on something or wish to know more on how
to use Twitter Cards check out [Twitter's Developer Documentation on Twitter
Cards](https://developer.twitter.com/en/docs/tweets/optimize-with-cards/overview/abouts-cards).
