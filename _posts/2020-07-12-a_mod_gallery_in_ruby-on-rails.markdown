---
layout: post
title:      "A Mod Gallery in Ruby-on-Rails"
date:       2020-07-12 13:57:43 -0400
permalink:  a_mod_gallery_in_ruby-on-rails
---


For my third project at Flatiron, I've decided to create a gallery that allows users to post mods for games, download and comment on other's mods, and search for mods by game and category. Users will be able to create their own account, or sign in through Github.

### Creating the Models
The models required for this app were more complicated than I first expected. The most important model was for Mods themselves. These would require a name, a description, the game that they modified, and several categories that they fit into. Rather than just store game as a string, let's create a Game model that the mods belong_to. This is more future-proof as we can now add content to the Game model as needed, such as the genre, what platforms it's available for, or whether it's on sale through some distributor.

I wanted to create Mods with many Categories so that Users could search more specifically. If a user wanted to replace environment models, it would be annoying to just search the model category and get all mods that replace any mesh. With a many-to-many relationship, users could search for "models" and "environment", possibly going deeper and looking for "interior".

Users require a username and password. There are several relationships between Users and Mods that are relevant. The most obvious is that mods belong_to the User who created them. Users can also download Mods; Users can download many Mods and Mods can have many Users who've downloaded them. Users should also be able to comment on mods. Using our typical ActiveRecord associations, we might think to do 
```
class Mod
    #the associations we've already established
    belongs_to :game
    has_many :mods_categories
    has_many :categories, through: :mods_categories
		
    #the user associations
    has_many :comments
    has_many :users, through: :comments
    has_many :users_downloads
    has_many :users, through: :users_downloads
    belongs_to :user
end
```
What behavior would we expect to result from having three associations to the user? To understand this we need to analyze what these lines of code actually do. The belongs_to and has_many keywords are macroprogramming that adds methods to call SQL statements based on our associations. So the first line connecting users would give us the method ```Mod#users``` which would return the users who had commented on the mod. But then the next line with ```has_many :users``` would overwrite ```Mod#users``` and return the users who had downloaded the mod. How do we manage multiple associations between the same two models?

One option would be to merge all of the join models. This UsersMods model would have foreign keys for the mod, the user, the creator's user_id, the commenter's user_id, the content of the comment, and any other information we might add like whether a user endorsed the mod. This would quickly get out of hand and create far too many rows with almost identical data.

Luckily ActiveRecord allows us [to give foreign_keys different names](https://stackoverflow.com/questions/13694654/specifying-column-name-in-a-references-migration)! The ```source:``` and ```class_name:``` keywords are additional arguments to has_many and belongs_to, and specify what model the foreign key is actually referencing. Thus we can rewrite the associations as follows:
```
class Mod
    # ...
    has_many :comments
    has_many :commenters, through: :comments, source: :user
    has_many :users_downloads
    has_many :downloads, through: :users_downloads, source: :user
    belongs_to :creator, class_name: "User"
end
```
[Source: is used for a has_many_through relationship](https://stackoverflow.com/questions/13611265/rails-difference-between-source-and-class-name-in-models#:~:text=%3Asource%20is%20used%20,has_many%20in%20the%20API%20here.), where class_name: works for the simpler association. Now we can call ```Mod#commenters``` to see who's commented, and still have access to ```Mod#downloads``` to see the users who've downloaded.

I made the mistake of over-shooting the mark with Rails generators, but luckily not by much. I used ```rails g resource``` for all of the models, but many of the join models don't require any views or other helpers, and a model generator would have been better suited for the task. 

### OAuth

I struggled a lot with Omniauth. I definitely procrastinated learning about it out of intimidation. Now that I've implemented it, that feels very silly because it is really no big deal at all. I kept having difficulties getting the routes to work, before finally realizing I was trying to direct the user to the url "/auth/github/callback" first. A little further reading on Omniauth and I figured it out: I was trying to send the user directly to the point where they were meant to return from Github. I made a new route without the /callback and things went smoothly from there.

### Nested Resources

All of my Join-Table-Fu above would come back to haunt me when trying to nest these resources in ways that made sense. For starters, when browsing mods, users would rarely want to see mods from different games in the same place. I wanted to make it possible to browse by category, but that would have to be nested within games, since a user isn't going to want to see all mods in the texture category, just the ones for the game they're currently playing. Thus I needed to nest at least an index page 2 levels deep (namely, games/id/categories/id/mod). I had to experiment with nests to discover that the nesting line (e.g. resources :games do) doesn't provide any routes for games, and another line (e.g. resources :games) is needed to fill out routes for that resource. 

I made two custom routes for download and endorse. These technically work on the UsersDownload join model, but since you can only create one and update it with your endorsement, and because these make most sense staying on the mod's show page, I kept them in the mods controller with two non-RESTful routes.

Making forms with Nested Resources was much easier than I was anticipating. I fell into a pretty good routine of setting up a draft of the form, checking what params it was giving, and fixing how things were named so that every piece of information could be fed to the correct model in a "model_params" private method and work right out of the box, no strings attached.

### Bootstrap and Rails/ERB

In order to make my site look a bit nicer I bundled in the Bootstrap gem. It's similar to Rails in that they're both guilty of doing The Absolute Most. For the most part I was able to look through free starter templates on w3schools and put something nice together. Forms were especially difficult because Bootstrap works in css classes applied to each element, where Rails uses ERB to create forms easily. Trouble is, you can't always easily give a class to an erb tag. I was stumped on how to leverage Bootstrap's much nicer-looking forms, until I discovered the Bootstrap-Form gem. This opens up the form_with blocks to much better styling, including validation errors displayed neatly under each input area. 

A few of my design ideas were a lot harder to implement. I really wanted to include box art of each game on the game index and show pages, but wasn't able to make it fit nicely on the cards, without being stretched awkwardly, or ruining the card deck tiling on different screen sizes. This also raised a lot of questions about the Rails Asset Pipeline, which we haven't really covered yet. I'm really hoping to learn more about html/css at some point, to figure out how to more confidently arrange things. I think for this project I Frankensteined together bits and pieces of different example code and templates to achieve the look I wanted, but going forward it would be nice to have a single, focused approach.

### The Biggest Missing Piece

But all of that aside, this app is missing something pretty serious! It's supposed to be a site where you can download and install mods, so where are all the actual mod files?! This will be the topic of my next out-of-class reading: how to allow users to upload content, scan to ensure that it's safe, store it somewhere, host it, and allow others to download it. There's definitely a lot to cover there, but I'm excited to learn more.

