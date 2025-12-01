"""
=======================================================
MERCIL DATABASE MODULE - Integration Package
For: https://github.com/trio-krittapas/Mercil-backend
=======================================================

วิธีใช้งาน:
1. วางไฟล์นี้ใน: mercil-backend/app/services/db_module.py
2. Update database.py ให้ใช้ functions จาก module นี้
3. เพิ่ม endpoints ใหม่ใน endpoints.py (ถ้าต้องการ)

หรือใช้แบบ standalone:
- วาง folder database_module/ ใน root ของ project
- Import ผ่าน: from database_module import GeospatialDB
"""

import logging
from typing import List, Dict, Optional, Tuple
from sentence_transformers import SentenceTransformer
from geopy.geocoders import Nominatim
import numpy as np
from sqlmodel import Session, select, func, text
from geoalchemy2.functions import ST_Distance, ST_DWithin, ST_MakePoint, ST_SetSRID
from app.services.database import Asset, AssetType  # ใช้ models ที่มีอยู่แล้ว

logger = logging.getLogger(__name__)


class MercilGeospatialDB:
    """
    Enhanced Geospatial Database Service for Mercil Backend
    
    เพิ่มความสามารถ:
    - Advanced hybrid search (semantic + spatial + filters)
    - Batch embedding generation
    - Geocoding automation
    - Statistical analysis
    """
    
    def __init__(self, db_session: Session, embedding_model: str = "paraphrase-multilingual-mpnet-base-v2"):
        """
        Initialize with existing database session
        
        Args:
            db_session: SQLModel database session
            embedding_model: Sentence transformer model name
        """
        self.db = db_session
        self.model = SentenceTransformer(embedding_model)
        self.geolocator = Nominatim(user_agent="mercil_proptech")
        logger.info(f"✅ MercilGeospatialDB initialized with {embedding_model}")
    
    # ==============================================
    # EMBEDDING FUNCTIONS
    # ==============================================
    
    def generate_embedding(self, text: str) -> List[float]:
        """
        Generate vector embedding from text
        
        Args:
            text: Input text (Thai/English)
            
        Returns:
            List of floats (768-dim for multilingual model)
        """
        embedding = self.model.encode(text, convert_to_numpy=True)
        return embedding.tolist()
    
    def batch_generate_embeddings(self, texts: List[str]) -> np.ndarray:
        """
        Generate embeddings for multiple texts efficiently
        
        Args:
            texts: List of text strings
            
        Returns:
            NumPy array of embeddings
        """
        embeddings = self.model.encode(texts, convert_to_numpy=True, show_progress_bar=True)
        return embeddings
    
    def update_asset_embedding(self, asset_id: int) -> bool:
        """
        Update embedding for a single asset
        
        Args:
            asset_id: Asset ID to update
            
        Returns:
            True if successful
        """
        try:
            asset = self.db.get(Asset, asset_id)
            if not asset:
                logger.warning(f"Asset {asset_id} not found")
                return False
            
            # Combine relevant text fields
            text_content = f"{asset.name_th or ''} {asset.name_en or ''} {asset.description or ''}".strip()
            
            if not text_content:
                logger.warning(f"Asset {asset_id} has no content for embedding")
                return False
            
            # Generate and update embedding
            embedding = self.generate_embedding(text_content)
            
            # Update using raw SQL (since embedding is stored as pgvector)
            query = text("""
                UPDATE assets 
                SET embedding = :embedding::vector 
                WHERE id = :asset_id
            """)
            self.db.execute(query, {"embedding": embedding, "asset_id": asset_id})
            self.db.commit()
            
            logger.info(f"✅ Updated embedding for asset {asset_id}")
            return True
            
        except Exception as e:
            logger.error(f"❌ Failed to update embedding for asset {asset_id}: {e}")
            self.db.rollback()
            return False
    
    def batch_update_embeddings(self, asset_ids: Optional[List[int]] = None) -> int:
        """
        Batch update embeddings for multiple assets
        
        Args:
            asset_ids: List of asset IDs (None = all assets)
            
        Returns:
            Number of assets updated
        """
        try:
            # Get assets to update
            query = select(Asset)
            if asset_ids:
                query = query.where(Asset.id.in_(asset_ids))
            
            assets = self.db.exec(query).all()
            
            if not assets:
                logger.warning("No assets to update")
                return 0
            
            updated_count = 0
            for asset in assets:
                if self.update_asset_embedding(asset.id):
                    updated_count += 1
            
            logger.info(f"✅ Updated {updated_count}/{len(assets)} asset embeddings")
            return updated_count
            
        except Exception as e:
            logger.error(f"❌ Batch embedding update failed: {e}")
            return 0
    
    # ==============================================
    # GEOCODING FUNCTIONS
    # ==============================================
    
    def geocode_address(self, address: str, city: Optional[str] = None, 
                       country: str = "Thailand") -> Optional[Tuple[float, float]]:
        """
        Convert address to coordinates
        
        Args:
            address: Street address
            city: City name
            country: Country name
            
        Returns:
            (latitude, longitude) or None
        """
        try:
            full_address = f"{address}, {city}, {country}" if city else f"{address}, {country}"
            location = self.geolocator.geocode(full_address)
            
            if location:
                logger.info(f"📍 Geocoded: {address} → ({location.latitude}, {location.longitude})")
                return (location.latitude, location.longitude)
            
            logger.warning(f"⚠️  Geocoding failed for: {address}")
            return None
            
        except Exception as e:
            logger.error(f"❌ Geocoding error: {e}")
            return None
    
    def update_asset_location(self, asset_id: int, 
                             lat: Optional[float] = None, 
                             lon: Optional[float] = None,
                             geocode_if_missing: bool = True) -> bool:
        """
        Update asset location coordinates
        
        Args:
            asset_id: Asset ID
            lat: Latitude (if None, will try geocoding)
            lon: Longitude (if None, will try geocoding)
            geocode_if_missing: Auto-geocode from address if coordinates not provided
            
        Returns:
            True if successful
        """
        try:
            asset = self.db.get(Asset, asset_id)
            if not asset:
                logger.warning(f"Asset {asset_id} not found")
                return False
            
            # Try geocoding if coordinates not provided
            if (lat is None or lon is None) and geocode_if_missing:
                address = asset.address or asset.name_th or asset.name_en
                if address:
                    coords = self.geocode_address(address)
                    if coords:
                        lat, lon = coords
            
            if lat is None or lon is None:
                logger.warning(f"No coordinates available for asset {asset_id}")
                return False
            
            # Update location using PostGIS
            asset.location_latitude = lat
            asset.location_longitude = lon
            
            self.db.commit()
            logger.info(f"✅ Updated location for asset {asset_id}: ({lat}, {lon})")
            return True
            
        except Exception as e:
            logger.error(f"❌ Failed to update location for asset {asset_id}: {e}")
            self.db.rollback()
            return False
    
    # ==============================================
    # ADVANCED SEARCH FUNCTIONS
    # ==============================================
    
    def semantic_search(self, query: str, limit: int = 10, 
                       min_similarity: float = 0.3) -> List[Dict]:
        """
        Pure semantic search using vector similarity
        
        Args:
            query: Search query text
            limit: Maximum results
            min_similarity: Minimum similarity threshold (0-1)
            
        Returns:
            List of asset dictionaries with similarity scores
        """
        try:
            # Generate query embedding
            query_embedding = self.generate_embedding(query)
            
            # Vector similarity search using pgvector
            sql_query = text("""
                SELECT 
                    id, asset_code, name_th, name_en, 
                    price, bedrooms, bathrooms,
                    location_latitude, location_longitude,
                    image_url,
                    1 - (embedding <=> :query_embedding::vector) AS similarity
                FROM assets
                WHERE embedding IS NOT NULL
                    AND (1 - (embedding <=> :query_embedding::vector)) >= :min_similarity
                ORDER BY embedding <=> :query_embedding::vector
                LIMIT :limit
            """)
            
            results = self.db.execute(
                sql_query, 
                {
                    "query_embedding": query_embedding, 
                    "min_similarity": min_similarity,
                    "limit": limit
                }
            ).fetchall()
            
            return [
                {
                    "id": r[0],
                    "asset_code": r[1],
                    "name_th": r[2],
                    "name_en": r[3],
                    "price": float(r[4]) if r[4] else None,
                    "bedrooms": r[5],
                    "bathrooms": r[6],
                    "latitude": float(r[7]) if r[7] else None,
                    "longitude": float(r[8]) if r[8] else None,
                    "image_url": r[9],
                    "similarity_score": float(r[10])
                }
                for r in results
            ]
            
        except Exception as e:
            logger.error(f"❌ Semantic search failed: {e}")
            return []
    
    def geospatial_search(self, lat: float, lon: float, 
                         radius_km: float = 10.0, 
                         limit: int = 20) -> List[Dict]:
        """
        Pure geospatial search within radius
        
        Args:
            lat: Center latitude
            lon: Center longitude
            radius_km: Search radius in kilometers
            limit: Maximum results
            
        Returns:
            List of assets with distance
        """
        try:
            radius_meters = radius_km * 1000
            
            sql_query = text("""
                SELECT 
                    id, asset_code, name_th, name_en,
                    price, bedrooms, bathrooms,
                    location_latitude, location_longitude,
                    image_url,
                    ST_Distance(
                        ST_SetSRID(ST_MakePoint(location_longitude, location_latitude), 4326)::geography,
                        ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography
                    ) AS distance_meters
                FROM assets
                WHERE location_latitude IS NOT NULL 
                    AND location_longitude IS NOT NULL
                    AND ST_DWithin(
                        ST_SetSRID(ST_MakePoint(location_longitude, location_latitude), 4326)::geography,
                        ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography,
                        :radius_meters
                    )
                ORDER BY distance_meters
                LIMIT :limit
            """)
            
            results = self.db.execute(
                sql_query,
                {"lat": lat, "lon": lon, "radius_meters": radius_meters, "limit": limit}
            ).fetchall()
            
            return [
                {
                    "id": r[0],
                    "asset_code": r[1],
                    "name_th": r[2],
                    "name_en": r[3],
                    "price": float(r[4]) if r[4] else None,
                    "bedrooms": r[5],
                    "bathrooms": r[6],
                    "latitude": float(r[7]) if r[7] else None,
                    "longitude": float(r[8]) if r[8] else None,
                    "image_url": r[9],
                    "distance_km": float(r[10]) / 1000 if r[10] else None
                }
                for r in results
            ]
            
        except Exception as e:
            logger.error(f"❌ Geospatial search failed: {e}")
            return []
    
    def advanced_hybrid_search(self, 
                              query: str,
                              lat: Optional[float] = None,
                              lon: Optional[float] = None,
                              radius_km: float = 10.0,
                              price_min: Optional[float] = None,
                              price_max: Optional[float] = None,
                              bedrooms_min: Optional[int] = None,
                              asset_type_ids: Optional[List[int]] = None,
                              limit: int = 20,
                              semantic_weight: float = 0.6,
                              spatial_weight: float = 0.4) -> List[Dict]:
        """
        Advanced hybrid search combining:
        - Semantic similarity
        - Geospatial proximity
        - Price filters
        - Attribute filters
        
        Args:
            query: Search query text
            lat, lon: Location for spatial search
            radius_km: Spatial search radius
            price_min, price_max: Price range filters
            bedrooms_min: Minimum bedrooms
            asset_type_ids: Filter by property types
            limit: Maximum results
            semantic_weight: Weight for semantic score (0-1)
            spatial_weight: Weight for spatial score (0-1)
            
        Returns:
            Ranked list of assets with combined scores
        """
        try:
            query_embedding = self.generate_embedding(query)
            radius_meters = radius_km * 1000
            
            # Build dynamic WHERE conditions
            where_conditions = ["embedding IS NOT NULL"]
            params = {
                "query_embedding": query_embedding,
                "limit": limit,
                "semantic_weight": semantic_weight,
                "spatial_weight": spatial_weight
            }
            
            # Price filters
            if price_min is not None:
                where_conditions.append("price >= :price_min")
                params["price_min"] = price_min
            
            if price_max is not None:
                where_conditions.append("price <= :price_max")
                params["price_max"] = price_max
            
            # Bedroom filter
            if bedrooms_min is not None:
                where_conditions.append("bedrooms >= :bedrooms_min")
                params["bedrooms_min"] = bedrooms_min
            
            # Asset type filter
            if asset_type_ids:
                where_conditions.append("asset_type_id = ANY(:asset_type_ids)")
                params["asset_type_ids"] = asset_type_ids
            
            # Spatial filter (if location provided)
            if lat is not None and lon is not None:
                where_conditions.append("""
                    ST_DWithin(
                        ST_SetSRID(ST_MakePoint(location_longitude, location_latitude), 4326)::geography,
                        ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography,
                        :radius_meters
                    )
                """)
                params["lat"] = lat
                params["lon"] = lon
                params["radius_meters"] = radius_meters
                
                # Build query with spatial scoring
                sql_query = text(f"""
                    WITH scored_assets AS (
                        SELECT 
                            id, asset_code, name_th, name_en,
                            price, bedrooms, bathrooms,
                            location_latitude, location_longitude,
                            image_url,
                            (1 - (embedding <=> :query_embedding::vector)) AS semantic_score,
                            (1 - (ST_Distance(
                                ST_SetSRID(ST_MakePoint(location_longitude, location_latitude), 4326)::geography,
                                ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography
                            ) / :radius_meters)) AS spatial_score
                        FROM assets
                        WHERE {" AND ".join(where_conditions)}
                    )
                    SELECT 
                        *,
                        (:semantic_weight * semantic_score + :spatial_weight * spatial_score) AS combined_score
                    FROM scored_assets
                    ORDER BY combined_score DESC
                    LIMIT :limit
                """)
            else:
                # Pure semantic search without spatial component
                sql_query = text(f"""
                    SELECT 
                        id, asset_code, name_th, name_en,
                        price, bedrooms, bathrooms,
                        location_latitude, location_longitude,
                        image_url,
                        (1 - (embedding <=> :query_embedding::vector)) AS semantic_score,
                        NULL AS spatial_score,
                        (1 - (embedding <=> :query_embedding::vector)) AS combined_score
                    FROM assets
                    WHERE {" AND ".join(where_conditions)}
                    ORDER BY combined_score DESC
                    LIMIT :limit
                """)
            
            results = self.db.execute(sql_query, params).fetchall()
            
            return [
                {
                    "id": r[0],
                    "asset_code": r[1],
                    "name_th": r[2],
                    "name_en": r[3],
                    "price": float(r[4]) if r[4] else None,
                    "bedrooms": r[5],
                    "bathrooms": r[6],
                    "latitude": float(r[7]) if r[7] else None,
                    "longitude": float(r[8]) if r[8] else None,
                    "image_url": r[9],
                    "semantic_score": float(r[10]) if r[10] else None,
                    "spatial_score": float(r[11]) if r[11] else None,
                    "combined_score": float(r[12]) if r[12] else None
                }
                for r in results
            ]
            
        except Exception as e:
            logger.error(f"❌ Advanced hybrid search failed: {e}")
            return []
    
    # ==============================================
    # STATISTICS & ANALYTICS
    # ==============================================
    
    def get_database_stats(self) -> Dict:
        """
        Get comprehensive database statistics
        
        Returns:
            Dictionary of statistics
        """
        try:
            stats = {}
            
            # Total assets
            stats["total_assets"] = self.db.exec(select(func.count(Asset.id))).one()
            
            # Assets with embeddings
            stats["assets_with_embeddings"] = self.db.exec(
                select(func.count(Asset.id)).where(Asset.embedding.is_not(None))
            ).one()
            
            # Assets with locations
            stats["assets_with_locations"] = self.db.exec(
                select(func.count(Asset.id))
                .where(Asset.location_latitude.is_not(None))
                .where(Asset.location_longitude.is_not(None))
            ).one()
            
            # Price statistics
            price_stats = self.db.exec(
                select(
                    func.min(Asset.price),
                    func.max(Asset.price),
                    func.avg(Asset.price)
                ).where(Asset.price.is_not(None))
            ).one()
            
            stats["price_min"] = float(price_stats[0]) if price_stats[0] else None
            stats["price_max"] = float(price_stats[1]) if price_stats[1] else None
            stats["price_avg"] = float(price_stats[2]) if price_stats[2] else None
            
            # Asset type distribution
            type_counts = self.db.exec(
                select(AssetType.name_en, func.count(Asset.id))
                .join(Asset, Asset.asset_type_id == AssetType.id)
                .group_by(AssetType.name_en)
            ).all()
            
            stats["asset_types"] = {name: count for name, count in type_counts}
            
            return stats
            
        except Exception as e:
            logger.error(f"❌ Failed to get database stats: {e}")
            return {}
    
    def get_location_coverage(self) -> Dict:
        """
        Analyze geographic coverage of assets
        
        Returns:
            Geographic statistics
        """
        try:
            coverage = {}
            
            # Bounding box of all assets
            bbox = self.db.exec(
                select(
                    func.min(Asset.location_latitude),
                    func.max(Asset.location_latitude),
                    func.min(Asset.location_longitude),
                    func.max(Asset.location_longitude)
                )
                .where(Asset.location_latitude.is_not(None))
                .where(Asset.location_longitude.is_not(None))
            ).one()
            
            if all(bbox):
                coverage["bbox"] = {
                    "lat_min": float(bbox[0]),
                    "lat_max": float(bbox[1]),
                    "lon_min": float(bbox[2]),
                    "lon_max": float(bbox[3])
                }
            
            # Average location (centroid)
            centroid = self.db.exec(
                select(
                    func.avg(Asset.location_latitude),
                    func.avg(Asset.location_longitude)
                )
                .where(Asset.location_latitude.is_not(None))
                .where(Asset.location_longitude.is_not(None))
            ).one()
            
            if all(centroid):
                coverage["centroid"] = {
                    "lat": float(centroid[0]),
                    "lon": float(centroid[1])
                }
            
            return coverage
            
        except Exception as e:
            logger.error(f"❌ Failed to get location coverage: {e}")
            return {}


# ==============================================
# INTEGRATION HELPER FUNCTIONS
# ==============================================

def get_geospatial_service(db_session: Session) -> MercilGeospatialDB:
    """
    Factory function to create geospatial service
    
    Usage in endpoints.py:
        geo_service = get_geospatial_service(db)
        results = geo_service.semantic_search("condo near BTS")
    """
    return MercilGeospatialDB(db_session)


def enhance_search_results(db_session: Session, query: str, 
                          filters: Dict, limit: int = 20) -> List[Dict]:
    """
    Enhanced search combining existing search with geospatial
    
    Drop-in replacement for existing search function
    
    Args:
        db_session: Database session
        query: Search query
        filters: Filter dictionary (price, bedrooms, location, etc.)
        limit: Result limit
        
    Returns:
        Enhanced search results
    """
    geo_service = MercilGeospatialDB(db_session)
    
    return geo_service.advanced_hybrid_search(
        query=query,
        lat=filters.get("latitude"),
        lon=filters.get("longitude"),
        radius_km=filters.get("radius_km", 10.0),
        price_min=filters.get("price_min"),
        price_max=filters.get("price_max"),
        bedrooms_min=filters.get("bedrooms_min"),
        asset_type_ids=filters.get("asset_type_ids"),
        limit=limit
    )


# ==============================================
# EXAMPLE USAGE
# ==============================================

"""
# In app/api/endpoints.py:

from app.services.db_module import get_geospatial_service, enhance_search_results

@app.post("/api/search/enhanced")
async def enhanced_search(
    request: SearchRequest,
    db: Session = Depends(get_db)
):
    geo_service = get_geospatial_service(db)
    
    results = geo_service.advanced_hybrid_search(
        query=request.query_text,
        lat=request.filters.get("latitude"),
        lon=request.filters.get("longitude"),
        price_min=request.filters.get("price_min"),
        price_max=request.filters.get("price_max"),
        limit=request.pagination.page_size
    )
    
    return {"results": results}

@app.get("/api/stats/database")
async def database_statistics(db: Session = Depends(get_db)):
    geo_service = get_geospatial_service(db)
    return geo_service.get_database_stats()

@app.post("/api/admin/update-embeddings")
async def update_all_embeddings(
    asset_ids: Optional[List[int]] = None,
    db: Session = Depends(get_db)
):
    geo_service = get_geospatial_service(db)
    count = geo_service.batch_update_embeddings(asset_ids)
    return {"updated": count}
"""
