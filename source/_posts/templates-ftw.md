---
title: Disentangling the Spaghetti Monster
date: 2023-02-24T16:53:22+02:00
tags:
  - C#
  - Gang Of Four
  - Design Patterns
  - Programming
categories:
  - Programming
  - Design Patterns
author: Michael Yarichuk
top_img: top.jpg
cover: cover.gif
---

## The What and Why

Picture this: you're working on a music streaming platform, and you already implemented live and offline playback, search functionality, and user ratings. The last piece of the puzzle? Playlist suggestions based on user preferences.
Seems simple, right? Just do some simple aggregations over likes and dislikes to figure out what genres of music the user is into, and serve up some recommendations based on that. So you decide to implement just that.

## The How

Let's assume our data is defined by the following entities.

```cs
public enum MusicGenre
{
    Rock,
    Classical,
    HipHop,
    // more genres omitted for clarity
}

//for each track with "like" or "dislike", create this
public record Vote(string TrackId, string UserId, bool IsUpvoted)
{
}

//Track is the most generic name I could think of :)
public record Track(
    string Id,
    string ArtistId,
    MusicGenre Genre, 
    string Name)
{
}
```

Now, let's see how will the implementation look like. First, for simplicity's sake, we will use Entity Framework and encapsulate access to track and votes data via the repository pattern.  

>Note: error and edge case handling are omitted for clarity and also, let's assume the calling code handles opening and commit/rollback of transactions as needed

```cs

//simplistic implementation of data access
public class TrackRepository : ITrackRepository 
{
  private readonly DbSet<Track> _tracks;
  //ctor here - omitted for clarity

  //fetch tracks specified by a list of track IDs
  public IReadOnlyList<Track> FetchByIDs(IEnumerable<string> trackIdsToFind)
  {
    //in this case, EF generates IN statement in the WHERE clause
    return _tracks.Where(track => trackIdsToFind.Contains(track.Id))
                  .ToList();
  }

  //fetch a list of random tracks filtered by specified genre list
  //note: we selected 100 as max fetch count arbitrarily, obviously this should be configurable
  public IReadOnlyList<Track> FetchRandomByGenres(IEnumerable<MusicGenre> genres, int maxTracksToFetch = 100)
  {
    //in this case, EF generates IN statement in the WHERE clause
    return _tracks.Where(track => genres.Contains(track.Genre))
                  .OrderBy(track => Guid.NewGuid()) //ensure randomness :)
                  .Take(maxTracksToFetch)
                  .ToList();
  }
}

public class VoteRepository: IVoteRepository
{
  private readonly DbSet<Vote> _votes;
  //ctor here - omitted for clarity

  public IReadOnlyList<Vote> FetchUpvotesFor(string userId)
  {
    return _votes.Where(vote => vote.UserId == userId && 
                                vote.IsUpvoted == true)
                 .ToList();
  }
}

```

Now, the class actually generating the playlist, would look something like this.

```cs
public class PlaylistRecommenderByGenrePreferences
{
  private ITrackRepository _trackRepo;
  private IVoteRepository _voteRepo;

  //ctor here - omitted for clarity

  public IEnumerable<Track> FetchRecommendationFor(string userId)
  {
    //first fetch all the votes and aggregate them
    var userVotes = _voteRepo.FetchUpvotesFor(userId);

    //fetch all upvoted tracks
    var upvotedTracks = _trackRepo.FetchByIDs(userVotes.Select(v => v.TrackId));

    //count how many times each track was upvoted
    var groupedByGenre = upvotedTracks
        .GroupBy(x => x.Genre)
        .Select(x => 
            new
            {
                Genre = x.Key, 
                Count = x.Count()
            })
        .OrderByDescending(x => x.Count);

    //take the first three most liked genres
    var mostLikedGenres = 
        groupedByGenre
            .Take(3)
            .Select(x => x.Genre);    

    //now get some random tracks for genres the user likes
    return _trackRepo.FetchRandomByGenres(mostLikedGenres);
  }
}
```

After we have a simple but functional implementation of random playlist generation is implemented, the new system goes live. Everything works well, and then, aftre a while, users another way to generate a playlist by artist that play the music user liked - sort of "more of the same".  

Okay, I might hear you say, that is not that hard. Simply aggregate artists that created the music users liked and fetch more from the same artists.
We can start from adding another method to ``TrackRepository``

```cs

//fetch some random tracks from the same artist
public IReadOnlyList<Track> FetchRandomByArtist(IEnumerable<string> artistIds)
{
  return _tracks.Where(track => artistIds.Contains(track.ArtistId))
                .OrderBy(Guid.NewGuid())
                .ToList();
}
```

Next piece of the puzzle would be to implement the new playlist generator. As we can see, the implementation would be similar to "by genre" playlist generator but different enough that we can't reuse the same method.

```cs

public class PlaylistRecommenderByArtistPreferences
{
  private ITrackRepository _trackRepo;
  private IVoteRepository _voteRepo;

  public PlaylistRecommenderByArtistPreferences(
    ITrackRepository trackRepo, 
    IVoteRepository voteRepo)
  {
    _trackRepo = trackRepo;
    _voteRepo = voteRepo;
  }

  public IEnumerable<Track> FetchRecommendationFor(string userId)
  {
    //first fetch all the votes and aggregate them
    var userVotes = _voteRepo.FetchUpvotesFor(userId);

    //fetch all upvoted tracks
    var upvotedTracks = _trackRepo.FetchByIDs(userVotes.Select(v => v.TrackId));

    //now aggregate by artistID
    var groupedByArtists = upvotedTracks
        .GroupBy(x => x.ArtistId)
        .Select(x => 
            new
            {
                ArtistId = x.Key, 
                Count = x.Count()
            })
        .OrderByDescending(x => x.Count);

    //take the first three most liked genress
    var mostLikedArtists = 
        groupedByArtists
            .Take(3)
            .Select(x => x.ArtistId);    
    
    return _trackRepo.FetchRandomByArtist(mostLikedArtists);
  }
}

```  

After implementing another type of playlist generation, users seem to be happy with the change. A week later, another feature request comes in: users now would like to see a "discovery playlist" generator, which would suggest artists and genres the user never upvoted before.  
How can something like this be approached? Well, you might decide to create another playlist generator class but by now I think you will agree with me that continuing in this way is not scalable in the long term. So, how can we approach this?  

## Code, meet Template Method Pattern

So what **is** this template pattern? It is basically a fancy way of saying "follow the same damn steps every time." It's like when you're making a sandwich, you always put the bread down first, then add some pastrami, Swiss cheese, lettuce, and whatever else veggies you happen to like. You don't start with the lettuce and end with the bread, that's just silly.

The Template Pattern is all about having a skeleton structure of an algorithm that stays the same, but you can implement the details as needed. So, let's say you're making different types of sandwiches, you still follow the same bread-meat-cheese-veggies routine, but you might switch out the pastrami for turkey or ham, or swap the "basic" cheese for something fancy like brie or gouda.

In we talk in code terms, it's simply having a class that defines some virtual or abstract methods that define the algorithm *steps*, which can be then customized by subclasses. Usually, such a "template" class has a bunch of abstract methods and a non-virtual method that defines the structure of the algorithm by using those abstract methods.

## So, show me the code

Let's apply the pattern to the code we have already written. First, let's define a base class.

```cs

public abstract class PlaylistGenerator
{

  public IEnumerable<Track> FetchRecommendationFor(string userId)
  {

  }
} 

```

## The When & Why

If we take a look at both of the implementations of the playlist generator, a pattern emerges: we have a certain steps that are recurring in both of the implementations and it makes sense that all other implementations would follow the same pattern.  

Conceptually, we can summarize a playlist generator like this:  

1. Fetch filtering and aggregation critera
2. Aggregate and fetch random tracks

If we see a pattern that emerges like this, we can leverage the Template Method pattern to make the code better structured and more maintainable.
