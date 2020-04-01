---
title:  How to cache user geolocation with Hazelcast and search for nearby users
author: xp
---
Hazelcast is great as a distributed object cache, it provides all the infrastructure as an in-memory key/value store for caching any type of objects. It even provides some query features to query based on object attributes, albeit the query feature is quite primitive as compared to what we have in relational database. But it does the work. And if it doesn't, you can extend it.

Let's say you are developing some kind of social app, and you are caching user information in Hazelcast, along with user's geolocation information. Now, you probably want to search for nearby users who are within a certain distance. How could you do a query like that in Hazelcast?

This post is going to describe how to search for nearby users in your Hazelcast cache, by implementing a new query predicate.

To keep thing simple in the example below, let's assume that you define your `CachedUser` class as follow:

```Java
public class MyCachedUser implements Serializable
{
    private static final long serialVersionUID = -2253075711571121144L;
    private String id = null;
    private double latitude;
    private double longitude;
    
    public MyCachedUser()
    {
        
    }
    
    public MyCachedUser(String id, double latitude, double longitude)
    {
        this.id = id;
        this.latitude = latitude;
        this.longitude = longitude;
    }
    
    public String getId()
    {
        return id;
    }
    public void setId(String id)
    {
        this.id = id;
    }
    public double getLatitude()
    {
        return latitude;
    }
    public void setLatitude(double l)
    {
        this.latitude = l;
    }
    public double getLongitude()
    {
        return longitude;
    }
    public void setLongitude(double l)
    {
        this.longitude = l;
    }
}
```

That is a very simple class for caching user information. We only have an ID, and the coordinates as latitude and longitude. Obviously, instead of implementing the default Java `Serializable` interface, you can also implement any of the serialization mechanism as provided in Hazelcast, or using any third-party serialization framework.

Before we go further, we need to be able to calculate the distance between two points, if you have the coordinates. I am not going into details explaining how to calculate that, you can find all the math formulas and detailed [explanation on Wikipedia](http://en.wikipedia.org/wiki/Geographical_distance). Here's a simple Java implementation:

```Java
public class GeoUtil
{
    public static final double EARTH_RADIUS = 6371.0d;
    
    public static double getDistance(double lat1, double lng1, double lat2, double lng2)
    {
        double dLat = toRadian(lat2 - lat1);
        double dLng = toRadian(lng2 - lng1);

        double a = Math.pow(Math.sin(dLat / 2), 2) + Math.cos(toRadian(lat1))
                * Math.cos(toRadian(lat2)) * Math.pow(Math.sin(dLng / 2), 2);

        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

        return EARTH_RADIUS * c; // returns result in kilometers
    }

    public static double toRadian(double degrees)
    {
        return (degrees * Math.PI) / 180.0d;
    }
}
```

Now that we have the required tools behind us, we can look at how to search for nearby users in Hazelcast. The query features in Hazelcast are not enough to make sophisticated searches. However, we can implement our own predicate to perform the searches that we want. Let's implement a predicate, called `GeoDistancePredicate`, which is used to calculate distance between two points when we perform the search:

```Java
public class GeoDistancePredicate implements Predicate<String, CachedUser>, DataSerializable
{
    private double latitude;
    private double longitude;
    private double distance;
    
    public GeoDistancePredicate()
    {
    }
    
    public GeoDistancePredicate(double lat, double lng, double dist)
    {
        this.latitude = lat;
        this.longitude = lng;
        this.distance = dist;
    }
    

    @Override
    public void readData(ObjectDataInput in) throws IOException
    {
        latitude = in.readDouble();
        longitude = in.readDouble();
        distance = in.readDouble();
    }

    @Override
    public void writeData(ObjectDataOutput out) throws IOException
    {
        out.writeDouble(latitude);
        out.writeDouble(longitude);
        out.writeDouble(distance);
    }

    @Override
    public boolean apply(Entry<String, CachedUser> entry)
    {
        boolean res = false;
        CachedUser u = entry.getValue();
        double dist = GeoUtil.getDistance(latitude, longitude, u.getLatitude(), u.getLongitude());
        res = (dist <= distance);
        return res;
    }

}
```

This class implements the `Predicate` interface. We pass in the GPS coordinates and the distance limit between the two points. As you can see, the most important code is in the `apply()` method, which is called to compare if a user is nearby, i.e. within the defined distance.

This class also implements the `DataSerializable` interface, as when you perform the searches, the predicate object will be sent to where the cached data are located, which might be on a different machine. Again, you can implement any serializable interface, as long as your object can be serialized and deserialized back.

Now that we have another piece of the puzzles in place, we can query nearby users as:

```Java
    Predicate predicate = new GeoDistancePredicate(lat, lng, dist);
    Collection usersFound = cachedUserMap.values(predicate);
```

That's it. Isn't this neat? You get back a list of users whose current location is within the defined distance.

Ok, but what if we want, not only the list of users, but also the distance of each user as well? For this, we need to do extra work, as there's no easy way to plug the requirement into the Hazelcast predicate and returned list. So we have to calculate the distance (again!) once we have the user list. Let's create another class called `DistancedCachedUser`, which contain the distance value and the cached user information:

```Java
public class DistancedCachedUser implements Comparable
{
    private CachedUser user;
    private double distance;
    
    public DistancedCachedUser(CachedUser user, double distance)
    {
        this.user = user;
        this.distance = distance;
    }
    
    public double getDistance()
    {
        return distance;
    }
    
    public CachedUser getUser()
    {
        return user;
    }

    @Override
    public int compareTo(DistancedCachedUser other)
    {
        if (this.distance < other.distance)
            return -1;
        else if (this.distance == other.distance)
            return 0;
        else
            return 1;
    }
}
```

This class also implements the `Comparable` interface, as we might want to sort the returned user list from the closest to the farthest.

Finally, here is the method to search for nearby users, and returned a list with the distance value for each user:

```Java
    public Collection getNearbyUsers(double lat, double lng, double dist)
    {
        ArrayList list = new ArrayList();
        Predicate predicate = new GeoDistancePredicate(lat, lng, dist);

        Collection usersFound = cachedUserMap.values(predicate);
        for (CachedUser u : usersFound)
        {
            DistancedCachedUser cu = new DistancedCachedUser(u, GeoUtil.getDistance(lat, lng, u.latitude, u.longitude));
            list.add(cu);
        }
        Collections.sort(list);
        return list;
    }
```

That's all. The last step was not as elegant and efficient as we might want, but it does the job.

Try it out, you can do a lot more by implementing your own predicates to perform the searches.

This is great. However, as soon as you have a lot of cached data, you would find that searching for nearby users is quite slow. What happened? In a next post, I'll show how to improve on this.
