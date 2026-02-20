# Geo

The Geo package provides a Geolocation and Mapping system for the Anchor Framework. It enables precise geocoding, distance calculations, and IP-based location services.

## Features

- **Multi-Driver Support**: Native integration with Google Maps and Nominatim (OpenStreetMap).
- **Proximity Search**: Find models within a specific radius using optimized SQL queries.
- **Fluent Trait**: "HasLocation" trait for seamless model-level geocoding and distance logic.
- **Distance Calculations**: Calculate differences between points in KM or Miles.
- **IP Geolocation**: Instantly resolve IP addresses to physical locations.
- **Reverse Geocoding**: Resolve Latitude/Longitude coordinates back to human-readable addresses.

## Installation

Geo is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Geo --packages
```

This will automatically:

- Register the `GeoServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/geo.php`

```php
return [
    'driver' => env('GEO_DRIVER', 'google'),

    'drivers' => [
        'google' => [
            'api_key' => env('GOOGLE_MAPS_API_KEY'),
        ],
        'nominatim' => [
            'user_agent' => env('APP_NAME'),
        ],
    ],

    'distance_unit' => 'km', // or 'miles'
];
```

## Basic Usage

### Geocoding & Distance

Use the `Geo` facade for core mapping operations:

```php
use Geo\Geo;

// Geocode Address
$location = Geo::geocode('1600 Amphitheatre Parkway, CA');
echo $location->latitude; // 37.4223

// Distance between points
$km = Geo::distance(37.42, -122.08, 40.71, -74.00);

// Resolve current visitor location
$location = Geo::locateIp();
```

## Model Integration

Add the `HasLocation` trait to any model with coordinates:

```php
use Geo\Traits\HasLocation;

class Store extends BaseModel
{
    use HasLocation;

    protected $latitudeColumn = 'lat';
    protected $longitudeColumn = 'lng';
}
```

### Proximity Searching

```php
// Find all stores within 10km of coordinates
$nearby = Store::withinRadius(37.42, -122.08, 10)->get();

// Calculate distance to another point
$dist = $store->distanceTo($userLat, $userLng);

// Auto-fill coordinates from address
$store->geocodeAddress('123 Main St, New York');
```

## Use Cases

### Geofenced Attendance

Ensure employees only check in when they are within 100 meters of their assigned office location.

#### Implementation

```php
use Geo\Geo;
use App\Models\Office;

public function checkIn(float $lat, float $lng)
{
    $office = Office::find($this->office_id);

    // Use the fluent distance API for better DX
    $distance = Geo::from($lat, $lng)->to($office)->unit('m')->get();

    if ($distance->isGreaterThan(100)) {
        throw new \Exception("Must be within 100m of office to check-in.");
    }

    // Proceed with check-in...
}
```

### Delivery Radius Priority

Automatically prioritize orders for customers within a 5km radius to ensure 30-minute delivery.

#### Implementation

```php
use Geo\Geo;
use App\Models\Order;

public function getPriorityOrders(float $storeLat, float $storeLng)
{
    // Find all 'pending' orders within 5km of the store
    return Order::where('status', 'pending')
        ->withinRadius($storeLat, $storeLng, 5.0)
        ->orderBy('created_at', 'asc')
        ->get();
}
```

## Package Integrations

### Audit Package (Activity Verification)

Location-based actions are logged with coordinates, allowing for retroactive verification of geofenced claims.

### Audit Package (Traceability)

The `Geo` package automatically integrates with the `Audit` package if it's installed. Every critical geo-operation is logged for security and analytics.

```php
use Audit\Audit;
use Geo\Models\Location;
use App\Models\User;

// No extra code needed! GeoManager handles this internally:
// 1. Geocoding requests are logged as 'geo.geocoded'
// 2. Reverse geocoding is logged as 'geo.reverse_geocoded'
// 3. IP lookups are logged as 'geo.ip_located'

// Custom logging with Geo context:
public function logCheckIn(User $user, Location $location)
{
    if (class_exists(Audit::class)) {
        Audit::make()
            ->event('user.check_in')
            ->on($user)
            ->metadata([
                'location' => $location->toArray(),
                'ip' => $_SERVER['REMOTE_ADDR']
            ])
            ->log();
    }
}
```

### Rollout Package (Geographic Targeting)

Feature flags can be targeted based on a user's `Location` metadata fluently.

```php
use Geo\Geo;
use Rollout\Rollout;

public function isPromoActive($user)
{
    $location = Geo::locateIp(request()->ip());

    // Better DX: Fluent check via Geo facade
    return Geo::isActive('city-wide-sale', $location);
}
```

## Advanced Features

### Driver Switching

Switch drivers on the fly for cost-efficiency or redundancy:

```php
// Force use of free Nominatim driver
Geo::driver('nominatim')->geocode('Paris, France');
```

### Performance & Analytics

Monitor API usage and resolution success rates:

```php
$analytics = Geo::analytics();

// 1. Track API usage metrics
$overview = $analytics->getOverview();
/**
 * Sample Data:
 * {
 *   "total_requests": 12500,
 *   "geocoding_success": 98.2,
 *   "cache_hit_rate": 45.5
 * }
 */

// 2. Growth Trends (Daily Volume)
$trends = $analytics->getDailyTrends(7);
/**
 * Sample Data:
 * [
 *   {"date": "2026-01-01", "requests": 340, "latency_ms": 120},
 *   {"date": "2026-01-02", "requests": 410, "latency_ms": 95}
 * ]
 */
```

## Service API Reference

### Geo (Facade)

| Method                       | Description                                     |
| :--------------------------- | :---------------------------------------------- |
| `geocode($address)`          | Resolves address to coordinates and metadata.   |
| `reverseGeocode($lat, $ln)`  | Resolves coordinates to human-readable address. |
| `distance($lat1, $ln1, ...)` | Calculates distance between two points.         |
| `locateIp($ip)`              | Resolves IP to geographic location.             |
| `driver($name)`              | Switches the active geolocation driver.         |
| `analytics()`                | Returns the `GeoAnalytics` service.             |

### HasLocation (Trait)

| Method                       | Description                                    |
| :--------------------------- | :--------------------------------------------- |
| `scopeWithinRadius($q, ...)` | Eloquent scope for proximity searching.        |
| `geocodeAddress($address)`   | Updates model coordinates based on the string. |
| `distanceTo($lat, $lng)`     | Returns distance from the model to the point.  |
| `hasCoordinates()`           | Checks if lat/lng are populated.               |

## Troubleshooting

| Error/Log        | Cause                          | Solution                                 |
| :--------------- | :----------------------------- | :--------------------------------------- |
| `OverQueryLimit` | API quota exceeded (Google).   | Increase quota or switch to `nominatim`. |
| `InvalidKey`     | Missing or incorrect API key.  | Check `.env` and `App/Config/geo.php`.   |
| "Zero Results"   | Address could not be resolved. | Verify address format and accuracy.      |

## Security Best Practices

- **API Key Restriction**: Ensure your Google Maps API key is restricted to your server's IP in the Google Cloud Console.
- **Privacy**: Never expose exact user coordinates in public APIs; consider rounding results or only providing distances.
- **Caching**: Cache geocoding results for frequently searched addresses to reduce API costs and improve performance.
