import json
import asyncio
from sentence_transformers import SentenceTransformer
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy import text
import os
from typing import List, Dict
from geopy.geocoders import Nominatim
from geopy.exc import GeocoderTimedOut, GeocoderServiceError
import time

# Database connection
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql+asyncpg://user:password@localhost/mercil_db")

engine = create_async_engine(DATABASE_URL, echo=True)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

# Initialize embedding model
model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')

# Initialize geocoder
geolocator = Nominatim(user_agent="mercil_geocoder")


async def init_db():
    """Initialize database with pgvector and PostGIS extensions"""
    async with engine.begin() as conn:
        # Enable extensions
        await conn.execute(text("CREATE EXTENSION IF NOT EXISTS vector;"))
        await conn.execute(text("CREATE EXTENSION IF NOT EXISTS postgis;"))
        
        # Create assets table
        await conn.execute(text("""
            CREATE TABLE IF NOT EXISTS assets (
                id SERIAL PRIMARY KEY,
                name TEXT NOT NULL,
                description TEXT,
                address TEXT,
                category TEXT,
                location GEOGRAPHY(POINT, 4326),
                embedding vector(384),
                metadata JSONB,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
        """))
        
        # Create index for vector similarity search
        await conn.execute(text("""
            CREATE INDEX IF NOT EXISTS assets_embedding_idx 
            ON assets USING ivfflat (embedding vector_cosine_ops)
            WITH (lists = 100);
        """))
        
        # Create spatial index
        await conn.execute(text("""
            CREATE INDEX IF NOT EXISTS assets_location_idx 
            ON assets USING GIST (location);
        """))
        
        print("✅ Database initialized successfully!")


def geocode_address(address: str, max_retries: int = 3) -> tuple:
    """Geocode address to lat/lon with retry logic"""
    for attempt in range(max_retries):
        try:
            location = geolocator.geocode(address, timeout=10)
            if location:
                return (location.latitude, location.longitude)
            else:
                print(f"⚠️ Could not geocode: {address}")
                return None
        except (GeocoderTimedOut, GeocoderServiceError) as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            print(f"❌ Geocoding failed for {address}: {e}")
            return None


async def load_assets_from_json(json_path: str):
    """Load assets from JSON file and store in database"""
    
    # Read JSON file
    with open(json_path, 'r', encoding='utf-8') as f:
        assets_data = json.load(f)
    
    print(f"📂 Loaded {len(assets_data)} assets from {json_path}")
    
    async with async_session() as session:
        for idx, asset in enumerate(assets_data, 1):
            try:
                # Extract data
                name = asset.get('name', '')
                description = asset.get('description', '')
                address = asset.get('address', '')
                category = asset.get('category', 'general')
                
                # Create text for embedding
                text_to_embed = f"{name}. {description}"
                
                # Generate embedding
                embedding = model.encode(text_to_embed).tolist()
                
                # Geocode address
                coords = geocode_address(address) if address else None
                
                # Prepare location for PostGIS (lon, lat format in WKT)
                location_wkt = f"POINT({coords[1]} {coords[0]})" if coords else None
                
                # Insert into database
                if location_wkt:
                    query = text("""
                        INSERT INTO assets (name, description, address, category, location, embedding, metadata)
                        VALUES (:name, :description, :address, :category, ST_GeogFromText(:location), :embedding, :metadata)
                    """)
                    
                    await session.execute(query, {
                        'name': name,
                        'description': description,
                        'address': address,
                        'category': category,
                        'location': location_wkt,
                        'embedding': embedding,
                        'metadata': json.dumps(asset)
                    })
                else:
                    # Insert without location
                    query = text("""
                        INSERT INTO assets (name, description, address, category, embedding, metadata)
                        VALUES (:name, :description, :address, :category, :embedding, :metadata)
                    """)
                    
                    await session.execute(query, {
                        'name': name,
                        'description': description,
                        'address': address,
                        'category': category,
                        'embedding': embedding,
                        'metadata': json.dumps(asset)
                    })
                
                print(f"✅ [{idx}/{len(assets_data)}] Inserted: {name}")
                
                # Rate limiting for geocoding
                if coords:
                    time.sleep(1)
                    
            except Exception as e:
                print(f"❌ Error processing {asset.get('name', 'Unknown')}: {e}")
                continue
        
        await session.commit()
        print(f"🎉 Successfully loaded {len(assets_data)} assets!")


async def search_similar_assets(query: str, limit: int = 5) -> List[Dict]:
    """Search for similar assets using vector similarity"""
    
    # Generate query embedding
    query_embedding = model.encode(query).tolist()
    
    async with async_session() as session:
        result = await session.execute(text("""
            SELECT 
                id,
                name,
                description,
                address,
                category,
                ST_Y(location::geometry) as latitude,
                ST_X(location::geometry) as longitude,
                1 - (embedding <=> :query_embedding) as similarity
            FROM assets
            ORDER BY embedding <=> :query_embedding
            LIMIT :limit
        """), {
            'query_embedding': query_embedding,
            'limit': limit
        })
        
        return [dict(row._mapping) for row in result]


async def main():
    """Main function to run the data loader"""
    print("🚀 Starting Mercil Data Loader...")
    
    # Initialize database
    await init_db()
    
    # Load assets from JSON
    json_path = "assets_rows.json"  # Update this path
    
    if os.path.exists(json_path):
        await load_assets_from_json(json_path)
    else:
        print(f"❌ File not found: {json_path}")
        print("Please create assets_rows.json with your data")
    
    # Test search
    print("\n🔍 Testing similarity search...")
    results = await search_similar_assets("โรงพยาบาล", limit=3)
    
    print("\nTop 3 similar assets:")
    for r in results:
        print(f"- {r['name']} (similarity: {r['similarity']:.3f})")


if __name__ == "__main__":
    asyncio.run(main())
