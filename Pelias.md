```c#
public (double lat, double lon) CalcularCentro((double lat, double lon) ponto1, (double lat, double lon) ponto2)
{
    double centroLat = (ponto1.lat + ponto2.lat) / 2;
    double centroLon = (ponto1.lon + ponto2.lon) / 2;

    return (centroLat, centroLon);
}
```
