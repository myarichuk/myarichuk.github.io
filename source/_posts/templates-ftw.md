---
title: Disentangling the Spaghetti Monster
date: 2023-03-18 08:28:00
tags:
  - C#
  - Gang Of Four
  - Design Patterns
  - Programming
categories:
  - Programming
  - Design Patterns
author: Michael Yarichuk
top_img: top.png
cover: cover.png
---

In this blog post, we will explore the practical application of a specific design pattern. To illustrate its usefulness, we will gradually reveal the problem in an "organic" manner, simulating how one might encounter such an issue in their daily programming tasks.

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
  //ctor omitted for clarity

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
  //ctor omitted for clarity

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
  private readonly ITrackRepository _trackRepo;
  private readonly IVoteRepository _voteRepo;

  //ctor omitted for clarity

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

After we have a simple but functional implementation of random playlist generation is implemented, the new system goes live. Everything works well, and then, after a while, users another way to generate a playlist by artist that play the music user liked - sort of "more of the same".  

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
  private readonly ITrackRepository _trackRepo;
  private readonly IVoteRepository _voteRepo;

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

So what **is** this template pattern? It is basically a fancy way of saying "follow the same damn steps every time." It's like when you're making a sandwich, you always put the bread down first, then add some pastrami, Swiss cheese, lettuce, and whatever veggies you happen to like. You don't start with the lettuce and end with the bread, that's just silly.

The Template Pattern is all about having a skeleton structure of an algorithm that stays the same, but you can implement the details as needed. So, let's say you're making different types of sandwiches, you still follow the same bread-meat-cheese-veggies routine, but you might switch out the pastrami for turkey or ham, or swap the "basic" cheese for something fancy like brie or gouda.

In we talk in code terms, it's simply having a class that defines some virtual or abstract methods that define the algorithm *steps*, which can be then customized by subclasses. Usually, such a "template" class has a bunch of abstract methods and a non-virtual method that defines the structure of the algorithm by using those abstract methods.

## So, show me the code

Let's apply the pattern to the code we have already written. First, let's add more "generalized" method to the track repository.

```cs

public IReadOnlyList<Track> FetchByPredicate(Func<Track, bool> filter)
{
  return _tracks.Where(track => filter(track))
                .ToList();
}

```

Now, let's define a base class that we will use as a "base" for our templates.

```cs

public abstract class PlaylistGenerator
{
  protected readonly ITrackRepository TrackRepo;
  protected readonly IVoteRepository VoteRepo;

  protected PlaylistGenerator(
    ITrackRepository trackRepo, 
    IVoteRepository voteRepo)
  {
    TrackRepo = trackRepo;
    VoteRepo = voteRepo;
  }

  protected abstract Func<Track, bool> PredicateFilter { get; }

  public IEnumerable<Track> FetchRecommendationFor(string userId) =>
     TrackRepo.FetchByPredicate(PredicateFilter);
} 

```

Alright, so now we've got a class that fetches recommendations, and we can mess around with how it fetches them by overriding the ``PredicateFilter``. But there is something missing: aggregation. If you take a look at the already implemented playlist generators, you would see the following pattern:

1. Fetch the tracks based on some critera (we just implemented it above)
2. Aggregate the tracks with "group by" clause and take some random tracks from the aggregated data

Sure, we could try to cram both (1) and (2) into the ``PredicateFilter``, but let's be honest, in a more complex code-base that's going to turn into a maintenance nightmare after enough time has passed. Instead, let's modify the abstract class ``PlaylistGenerator`` to include the second stage of the algorithm:

```cs

public abstract class PlaylistGenerator<TAggregationKey>
{
  protected readonly ITrackRepository TrackRepo;
  protected readonly IVoteRepository VoteRepo;
  protected readonly int MaxTrackGroupsToTake;
  protected readonly int MaxResults;

  protected PlaylistGenerator(
    ITrackRepository trackRepo, 
    IVoteRepository voteRepo,
    int maxTrackGroupsToTake = 3,
    int maxResults = 100)
  {
    TrackRepo = trackRepo;
    VoteRepo = voteRepo;
    MaxTrackGroupsToTake = maxTrackGroupsToTake;
    MaxResults = maxResults;
  }

  protected abstract Func<Track, bool> PredicateFilter { get; }

  protected abstract Func<Track, TAggregationKey> AggregationKey { get; }

  public IEnumerable<Track> FetchRecommendationFor(string userId)
  {
     var tracksToAggregate = VoteRepo.FetchByPredicate(PredicateFilter);
     var tracksOfGroupedTracks = tracksToAggregate
       .GroupBy(AggregationKey)
       .Take(MaxTrackGroupsToTake)
       .SelectMany(x => x.Select(g => g.TrackId));

     var results = TrackRepo
       .FetchByIDs(tracksOfGroupedTracks)
       .OrderBy(Guid.NewGuid())
       .Take(MaxResults)
       .ToList();    

     return results;   
  }
} 

```

As we can see, in the ``FetchRecommendationFor`` implementation above, we have the algorithm structure unchanging but it can be influenced by override of an abstract properties. Now, all we have left is to implement ``PlaylistRecommenderByArtistPreferences`` and ``PlaylistRecommenderByGenrePreferences``.  

Let's start from ``PlaylistRecommenderByGenrePreferences``:

```cs
public class PlaylistRecommenderByGenrePreferences: PlaylistGenerator<MusicGenre>
{
  private readonly string _targetUserId;

  public PlaylistRecommenderByGenrePreferences(
    ITrackRepository trackRepo, 
    IVoteRepository voteRepo,
    string targetUserId) : base(trackRepo, voteRepo)
  {
    _targetUserId = targetUserId;
  }
  
  protected override Func<Track, bool> PredicateFilter
  {
    get
    {      
      var userVotes = VoteRepo.FetchUpvotesFor(_targetUserId);   
      var trackIds = userVotes.Select(v => v.TrackId).ToList();

      return track => trackIds.Contains(track.Id);
    }
  }

  protected override Func<Track, MusicGenre> AggregationKey => 
    (track) => track.Genre;
}
```

Implementing ``PlaylistRecommenderByArtistPreferences`` would not be much harder:

```cs
public class PlaylistRecommenderByArtistPreferences: PlaylistGenerator<string>
{
  private readonly string _targetUserId;

  public PlaylistRecommenderByGenrePreferences(
    ITrackRepository trackRepo, 
    IVoteRepository voteRepo,
    string targetUserId) : base(trackRepo, voteRepo)
  {
    _targetUserId = targetUserId;
  }
  
  protected override Func<Track, bool> PredicateFilter
  {
    get
    {      
      var userVotes = VoteRepo.FetchUpvotesFor(_targetUserId);    
      var trackIds = userVotes.Select(v => v.TrackId).ToList();

      return track => trackIds.Contains(track.Id);
    }
  }

  protected override Func<Track, string> AggregationKey => 
    (track) => track.ArtistId;  
}
```

## So, let's review our epic battle with the Spaghetti Monster

We've fought with the Spaghetti Monster and emerged victorious. By utilizing the Template Method Pattern, we've brought that messy code closer to a well-organized and maintainable piece of art. And yes, I might be exaggerating a bit, but you get the idea.

We have improved:

1. Clarity: Our code is now more readable and easier to understand. No more tangled noodles to decipher!
2. Stability: By separating the logic into different methods, we've reduced the risk of unwanted side effects.
3. Maintainability: Future updates and changes will be much easier, as each method has a clear purpose and responsibility.

So, the next time you find yourself lost in the depths of bad code, remember that the Template Method Pattern may help to disentangle at least *some* of the mess. May the clean code be with you, and may your sandwiches always be delicious!
