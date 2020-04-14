---
title:  Improvement on user geolocation cache with Hazelcast
author: xp
tags: Programming, Java, Hazelcast
---
In the last post, we have seen how to cache user geolocation data in Hazelcast, and search for nearby users. This was great. However, as soon as you have a lot of cached data, you would find that searching for nearby users is quite slow. What was wrong?

As you have remembered, we have implemented a `GeoDistancePredicate` which computes to see if a user is within the distance limit to a specific point. The way this predicate works is that, for every entry cached in memory, Hazelcast would invoke the `apply()` method to see if the entry satisfies the distance limit criterion. If you have cached a large data set, this method will loop through all entries in the whole data set, one by one. Therefore, this is basically a _O(n)_ problem. There's no way it can be fast, if you have a large data set.

We need to find a better way to quickly search for nearby users, using only the mechanisms available in Hazelcast.

Hazelcast defined two kinds of predicates, one is the normal `Predicate`, and the other is called `IndexAwarePredicate`. As the name implies, the second predicate interface will use the internal attribute indexes of your cached object to speed up query. In the previous post, our `GeoDistancePredicate` only implements the `Predicate` interface, therefore, during the query operation, it has to scan through the whole data set to get the right results. In this post, we are going to change the implementation to take into consideration indexing, which should significantly help with search performance.

Before we can use the indexes for searching, we have to tell Hazelcast which attributes to index. Remember in the last post, we had defined a class called `MyCachedUser`. This simple class has three attributes, but when we search for nearby users, we are only interested in the user's location, namely their latitude and longitude coordinates. Therefore, we want to build indexes on these two attributes to help speeding up with searches.

At start up, we need to tell Hazelcast that we want to index these attributes, such as:

```Java
    HazelcastInstance hz = Hazelcast.newHazelcastInstance();
    IMap<Object, Object> map = hz.getMap("users");
    map.addIndex("latitude", true);
    map.addIndex("longitude", true);
```

This tells Hazelcast that we want the two attributes to be indexed, and that we will search for ranges, hence, we set the second parameter to `true`. This tells Hazelcast that these two attributes need to be indexed and sorted.

Now that we have the indexes in place, we can use the indexes to limit the search space to a specific range. Given a point, we want to draw a circle with the point as the center, with the distance limit as the radius of the circle. And we want to limit our search within the circle only. Before we can do that, we need to figure out how to draw the circle first, then find all users whose current location is within the circle.

However, drawing a circle would not allow us to take advantage of the latitude/longitude indexes. It is much easier to draw a square first, and limit the search ranges within the square. We know that the circle is within the square. After we have found all users within the square, we can eliminate those who are not within the circle, mainly, the users found at the four corners of the square.

To draw the square, we need a point to the north of the central point, a point to the east, a point to the south, and a point to the west, all with distance equal to the distance limit we want to search for.

For these four points, we have a starting point, the bearing, and the distance. The formula to find the point is well-known, so I'm not going into all the details. The Java implementation is followed:

```Java
    public static GeoCoordinate fromBearingDistance(double lat, double lon, double bearing, double d)
    {
        /*
         * φ2 = asin( sin(φ1)*cos(d/R) + cos(φ1)*sin(d/R)*cos(θ) )
         * λ2 = λ1 + atan2( sin(θ)*sin(d/R)*cos(φ1), cos(d/R)−sin(φ1)*sin(φ2))
         */
        double lat1 = Math.toRadians(lat);
        double lon1 = Math.toRadians(lon);
        double brng = Math.toRadians(bearing);
        double lat2 = Math.asin(Math.sin(lat1) * Math.cos(d / EARTH_RADIUS)
                + Math.cos(lat1) * Math.sin(d / EARTH_RADIUS) * Math.cos(brng));
        double lon2 = lon1
                + Math.atan2(
                        Math.sin(brng) * Math.sin(d / EARTH_RADIUS)
                                * Math.cos(lat1), Math.cos(d / EARTH_RADIUS)
                                - Math.sin(lat1) * Math.sin(lat2));
        return new GeoCoordinate(Math.toDegrees(lat2), Math.toDegrees(lon2));
    }
```

The coordinates are provided in degrees, but the formula calculates based on radians. Therefore, we need to convert to radians first, and convert the result back to degrees. And the `GeoCoordinate` is defined as:

```Java
public class GeoCoordinate implements Portable
{
    public static final String KEY_LATITUDE          = "latitude";
    public static final String KEY_LONGITUDE         = "longitude";

    private double latitude;
    private double longitude;

    public GeoCoordinate()
    {
    }

    public GeoCoordinate(double lat, double lng)
    {
        this.latitude = lat;
        this.longitude = lng;
    }

    public double getLatitude()
    {
        return latitude;
    }

    public void setLatitude(double lat)
    {
        this.latitude = lat;
    }

    public double getLongitude()
    {
        return longitude;
    }
    public void setLongitude(double lng)
    {
        this.longitude = lng;
    }

    @Override
    public int getFactoryId()
    {
        return CachedObjectFactory.FACTORY_ID;
    }

    @Override
    public int getClassId()
    {
        return CachedObjectFactory.TYPE_GEOCOORDINATE;
    }

    @Override
    public void writePortable(PortableWriter writer) throws IOException
    {
        writer.writeDouble(KEY_LATITUDE, latitude);
        writer.writeDouble(KEY_LONGITUDE, longitude);
    }

    @Override
    public void readPortable(PortableReader reader) throws IOException
    {
        latitude = reader.readDouble(KEY_LATITUDE);
        longitude = reader.readDouble(KEY_LONGITUDE);
    }

    @Override
    public String toString()
    {
        return "lat=" + latitude + ";lng=" + longitude;
    }
}
```

Now, it's time to revisit our `GeoLocationPredicate` implementation. As we said earlier, we need to make this predicate aware of the indexes. We need to implement the `IndexAwarePredicate` interface, or we can derive it from the class `AbstractPredicate`, which also implements the `IndexAwarePredicate` interface. Without further ado, here is the modified version of `GeoLocationPredicate`:

```Java
public class GeoDistancePredicate extends AbstractPredicate
{
    private double latitude;
    private double longitude;
    private double distance;
    private double latFloor;
    private double latCeiling;
    private double lngFloor;
    private double lngCeiling;

    public GeoDistancePredicate()
    {
        super("latitude");
    }

    public GeoDistancePredicate(double lat, double lng, double dist)
    {
        super("latitude");
        this.latitude = lat;
        this.longitude = lng;
        this.distance = dist;

        init();
    }

    private void init()
    {
        GeoCoordinate c = GeoUtil.fromBearingDistance(latitude, longitude, GeoUtil.NORTH, distance);
        latCeiling = c.getLatitude();
        c = GeoUtil.fromBearingDistance(latitude, longitude, GeoUtil.SOUTH, distance);
        latFloor = c.getLatitude();
        c = GeoUtil.fromBearingDistance(latitude, longitude, GeoUtil.EAST, distance);
        lngCeiling = c.getLongitude();
        c = GeoUtil.fromBearingDistance(latitude, longitude, GeoUtil.WEST, distance);
        lngFloor = c.getLongitude();
    }

    @Override
    public void readData(ObjectDataInput in) throws IOException
    {
        super.readData(in);
        latitude = in.readDouble();
        longitude = in.readDouble();
        distance = in.readDouble();

        init();
    }

    @Override
    public void writeData(ObjectDataOutput out) throws IOException
    {
        super.writeData(out);
        out.writeDouble(latitude);
        out.writeDouble(longitude);
        out.writeDouble(distance);
    }

    @Override
    public boolean apply(Entry entry)
    {
        boolean res = false;
        Object obj = entry.getValue();
        if (obj instanceof MyCachedUser)
        {
            MyCachedUser u = (MyCachedUser) obj;
            double dist = GeoUtil.getDistance(latitude, longitude, u.getLatitude(), u.getLongitude());
            res = (dist &lt;= distance);
        }
        return res;
    }

    @Override
    public Set filter(QueryContext queryContext)
    {
        String sql = "latitude BETWEEN " + latFloor + " AND " + latCeiling + " AND " + "longitude BETWEEN " + lngFloor + " AND " + lngCeiling;
        SqlPredicate sqlPred = new SqlPredicate(sql);
        Set entries = sqlPred.filter(queryContext);
        Set endList = new HashSet();
        if (logger.isDebugEnabled())
        {
            for (QueryableEntry e : entries)
            {
                Object v = e.getValue();
                if (v instanceof MyCachedUser)
                {
                    MyCachedUser u = (MyCachedUser) v;
                    double dist = GeoUtil.getDistance(latitude, longitude, u.getLatitude(), u.getLongitude());
                    if (dist &lt;= distance)
                    {
                        endList.add(e);
                    }
                }
            }
        }
        return endList;
    }

}
```

As you can see, given a central point, we find out the points at north, at east, at south and at west. Using the coordinates of the four points to create a square, we then limit the search space within that square only. The bearings are defined as:

``` java
    public static final double NORTH = 0.0d;
    public static final double EAST = 90.0d;
    public static final double SOUTH = 180.0d;
    public static final double WEST = 270.0d;
```

In this new implementation, the method `apply()` will not be useful anymore, but the method `filter()` will be called to filter results based on our new search criteria. Here, we use the `Between` and the `And` predicates to search for all users within the square, then we filter out all those whose current location is not within the circle.

That's it, there will be no change to your application logic. With this new modification, we reduce the search complexity from _O(n)_ to _O(log n)_, which should be significantly faster than the previous implementation.
