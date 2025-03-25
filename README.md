import { HttpClient, withJsonpSupport } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { environment } from '@env';
import { Observable } from 'rxjs';
import { catchError, tap } from 'rxjs/operators';

interface cacheItem {
  value: any;
  expiry: number;
}

@Injectable({
  providedIn: 'root'
})
export class SponsoredCarService {

  private baseUrl = environment.baseUrl;
  constructor(private httpClient: HttpClient) { }

  //private cache = new Map<string, any>();

  private cache = new Map<string, cacheItem>();
  private defaultExpiration = 120000; //2 min expiration time for testing.

  SaveSponsoredVehicleInformation(vehicleVersionDTO: any) {
    const formattedDate = new Date(vehicleVersionDTO.endDateTime).toISOString();
    const requestBody = {
      "vehicleVersionId": vehicleVersionDTO.vehicleVersionId,
      "endDateTime": formattedDate
    };

    return this.httpClient.post<string>(
      this.baseUrl + '/Vehicle' + '/SaveSponsoredVehicleInformation/', requestBody)
  }

  DeleteSponsoredVehicleInformation(vehicleVersionDTO: any) {
    const body = { "vehicleVersionId": vehicleVersionDTO };
    return this.httpClient.post<string>(
      this.baseUrl + '/Vehicle' + '/DeleteSponsoredVehicleInformation/', body)
  }

  AllVersionByMakeAndByModel(vehicleMakeId: Number, vehicleModelId: Number) {
    return this.httpClient.get(this.baseUrl + '/Vehicle' + '/AllVersionByMakeAndByModel/' + vehicleMakeId + '/' + vehicleModelId);
  }

  AllModelsByMake(vehicleMakeId: Number) {
    const payload = {};
    return this.httpClient.get(this.baseUrl + '/Vehicle' + '/AllModelsByMake/' + vehicleMakeId);
  }

  AllMakesByCountry() {
    return this.httpClient.post(this.baseUrl + '/Vehicle' + '/AllMakesByCountry/', {});
  }

  BestDealVehicles() {
    return this.httpClient.get(this.baseUrl + '/Vehicle' + '/BestDealVehicles');
  }

  SetLocalStorageMakes(res: any) {
    const expiry = new Date().getTime() + this.defaultExpiration;
    const cacheItem: cacheItem = { value: res, expiry };
    this.cache.set('makes', cacheItem);
  }

  getLocalStorage() {
    const data = this.cache.get('makes');
    if (!data) {
      return undefined;
    }

    if (new Date().getTime() > data.expiry) {
      this.cache.delete('makes');
      return undefined;
    }

    return data.value;
  }

  clearCache() {
    localStorage.removeItem('cachedModels');
  }

  getAllModelsByMake(responseKey: string, vehicleMakeId: Number): Observable<any> {
    const cachedData = this.cache.get('allModelsByMake');

    let responses = cachedData?.value ? cachedData.value : {};

    if(cachedData)
    {
      if(responses[responseKey])
      {
        if (new Date().getTime() > cachedData.expiry) {
          this.cache.delete('allModelsByMake');
        }
      }
    }

    if (responses[responseKey]) {
      return new Observable(observer => {
        observer.next(responses[responseKey]);
        observer.complete();
      });
    } else {
      return this.httpClient.get(this.baseUrl + '/Vehicle' + '/AllModelsByMake/' + vehicleMakeId).pipe(
        catchError(err => {
          console.error('API error:', err);
          throw err;
        }),
        tap(data => {
          responses[responseKey] = data;
          const expiry = new Date().getTime() + this.defaultExpiration;
          const cacheItem: cacheItem = { value: responses, expiry };
          this.cache.set('allModelsByMake', cacheItem);
        })
      );
    }
  }

  getAllVersionsByMakeAndModel(responseKey: string, vehicleMakeId: Number, vehicleModelId: Number): Observable<any> {
    const cachedData = this.cache.get('allVersionsByMakeAndModel');
    
    //let responses = cachedData ? JSON.parse(cachedData.value) : {};
    let responses = cachedData?.value ? cachedData.value : {};

    if(cachedData)
      {
        if(responses[responseKey])
        {
          if (new Date().getTime() > cachedData.expiry) {
            this.cache.delete('allVersionsByMakeAndModel');
          }
        }
      }

    // if (new Date().getTime() > responses[responseKey].expiry) {
    //   this.cache.delete(responses[responseKey]);
    // }

    if (responses[responseKey]) {
      return new Observable(observer => {
        observer.next(responses[responseKey]);
        observer.complete();
      });
    } else {
      return this.httpClient.get(this.baseUrl + '/Vehicle' + '/AllVersionByMakeAndByModel/' + vehicleMakeId + '/' + vehicleModelId).pipe(
        catchError(err => {
          console.error('API error:', err);
          throw err;
        }),
        tap(data => {
          responses[responseKey] = data;

          const expiry = new Date().getTime() + this.defaultExpiration;
          const cacheItem: cacheItem = { value: responses, expiry };
          this.cache.set('allVersionsByMakeAndModel', cacheItem);
        })
      );
    }
  }

  clearCacheData(): void {
    localStorage.removeItem('allModelsByMake');
    localStorage.removeItem('allVersionsByMakeAndModel');
  }
}
