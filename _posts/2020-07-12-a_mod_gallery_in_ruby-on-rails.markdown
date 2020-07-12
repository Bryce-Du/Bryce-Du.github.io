---
layout: post
title:      "A Mod Gallery in Ruby-on-Rails"
date:       2020-07-12 17:57:42 +0000
permalink:  a_mod_gallery_in_ruby-on-rails
---


For my third project at Flatiron, I've decided to create a gallery that allows users to post mods for games, download and comment on other's mods, and search for mods by game and category. Users will be able to create their own account, or sign in through $$$TODO$$$.

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



