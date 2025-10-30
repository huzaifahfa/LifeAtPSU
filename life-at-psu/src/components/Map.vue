<script setup>
import { ref, onMounted } from 'vue';
import "leaflet/dist/leaflet.css"
import * as L from 'leaflet';
import { eventMarkers } from '../data/markers.js';

const initialMap = ref(null);

onMounted(()=> {
    var osm = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
        maxZoom: 18,
        attribution: '© OpenStreetMap'
    });

    var osmHOT = L.tileLayer('https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png', {
        maxZoom: 18,
        attribution: '© OpenStreetMap contributors'});

    var Esri_WorldImagery = L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}', {
        attribution: 'Tiles &copy; Esri &mdash; Source: Esri, i-cubed, USDA, USGS, AEX, GeoEye, Getmapping, Aerogrid, IGN, IGP, UPR-EGP, and the GIS User Community'
    });

    initialMap.value = L.map('map', {zoomControl: false}).setView([40.7982,-77.8599], 17);
    L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
        minZoom: 17,
        maxZoom: 18, 
        attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
        maxBoundsViscosity: 1.0,
        tileSize: 256,
        reuseTiles: true, 
        unloadInvisibleTiles: true
    }).addTo(initialMap.value);

    // Set expanded bounds to cover the entire Penn State University Park campus including all facilities
    var southWest = L.latLng(40.7750, -77.8900),
        northEast = L.latLng(40.8150, -77.8300);
    var bounds = L.latLngBounds(southWest, northEast);

    initialMap.value.setMaxBounds(bounds);
    initialMap.value.on('drag', function() {
        initialMap.value.panInsideBounds(bounds, { animate: false });
    });

    // Add event markers that open custom overlay galleries on click
    eventMarkers.forEach((marker, index) => {
        const leafletMarker = L.marker(marker.coordinates).addTo(initialMap.value);
        
        // Add click event to open custom overlay gallery
        leafletMarker.on('click', function() {
            if (marker.images && marker.images.length > 0) {
                openCustomGallery(marker, index);
            } else {
                // Show simple popup if no images
                leafletMarker.bindPopup(`<b>${marker.title}</b><br>${marker.description}<br><small>No images available</small>`)
                    .openPopup();
            }
        });
        
        // Add hover tooltip
        leafletMarker.bindTooltip(`<b>${marker.title}</b>`, {
            permanent: false,
            direction: 'top',
            offset: [0, -10]
        });
    });

    // Custom gallery overlay function
    function openCustomGallery(marker, markerIndex) {
        // Remove any existing gallery overlay
        const existingOverlay = document.querySelector('.custom-gallery-overlay');
        if (existingOverlay) {
            existingOverlay.remove();
        }

        // Create overlay container
        const overlay = document.createElement('div');
        overlay.className = 'custom-gallery-overlay';
        overlay.innerHTML = `
            <div class="gallery-backdrop" onclick="closeCustomGallery()"></div>
            <div class="gallery-container">
                <div class="gallery-header">
                    <h3>${marker.title}</h3>
                    <button class="close-btn" onclick="closeCustomGallery()">×</button>
                </div>
                <div class="gallery-content">
                    <div class="main-image-container">
                        <img id="main-image" src="${marker.images[0]}" alt="${marker.title}">
                        <div class="nav-buttons">
                            <button class="nav-btn prev-btn" onclick="changeImage(-1)">‹</button>
                            <button class="nav-btn next-btn" onclick="changeImage(1)">›</button>
                        </div>
                    </div>
                    <div class="thumbnails">
                        ${marker.images.map((img, idx) => 
                            `<img src="${img}" alt="Thumbnail ${idx + 1}" class="thumbnail ${idx === 0 ? 'active' : ''}" onclick="setMainImage(${idx})">`
                        ).join('')}
                    </div>
                </div>
                <div class="image-counter">
                    <span id="current-image">1</span> / ${marker.images.length}
                </div>
            </div>
        `;

        // Add overlay to map container (not body)
        const mapContainer = document.getElementById('map');
        mapContainer.appendChild(overlay);

        // Store current gallery data
        window.currentGallery = {
            images: marker.images,
            currentIndex: 0,
            title: marker.title
        };
    }

    // Global functions for gallery navigation
    window.closeCustomGallery = function() {
        const overlay = document.querySelector('.custom-gallery-overlay');
        if (overlay) {
            overlay.remove();
        }
        window.currentGallery = null;
    };

    window.changeImage = function(direction) {
        if (!window.currentGallery) return;
        
        const gallery = window.currentGallery;
        gallery.currentIndex += direction;
        
        if (gallery.currentIndex < 0) {
            gallery.currentIndex = gallery.images.length - 1;
        } else if (gallery.currentIndex >= gallery.images.length) {
            gallery.currentIndex = 0;
        }
        
        updateGalleryDisplay();
    };

    window.setMainImage = function(index) {
        if (!window.currentGallery) return;
        window.currentGallery.currentIndex = index;
        updateGalleryDisplay();
    };

    function updateGalleryDisplay() {
        if (!window.currentGallery) return;
        
        const gallery = window.currentGallery;
        const mainImage = document.getElementById('main-image');
        const currentImageSpan = document.getElementById('current-image');
        const thumbnails = document.querySelectorAll('.thumbnail');
        
        if (mainImage) {
            mainImage.src = gallery.images[gallery.currentIndex];
            mainImage.alt = `${gallery.title} - Image ${gallery.currentIndex + 1}`;
        }
        
        if (currentImageSpan) {
            currentImageSpan.textContent = gallery.currentIndex + 1;
        }
        
        thumbnails.forEach((thumb, idx) => {
            thumb.classList.toggle('active', idx === gallery.currentIndex);
        });
    }

    var baseMaps = {
        "OpenStreetMap": osm,
        "OpenStreetMap.HOT": osmHOT,
        "Esri_WorldImagery": Esri_WorldImagery
    };

    L.control.layers(baseMaps).addTo(initialMap.value);
});

</script>


<template>
  <div id="map" style="height:100vh;"></div>
</template>

<style>
/* Custom Gallery Overlay Styling */
.custom-gallery-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
}

.gallery-backdrop {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.419);
  opacity: 1.0;
  cursor: pointer;
}

.gallery-container {
  position: relative;
  background: white;
  border-radius: 16px;
  box-shadow: 0 25px 60px rgba(0, 0, 0, 0.3);
  max-width: 90%;
  max-height: 85%;
  width: 700px;
  overflow: hidden;
  z-index: 1001;
}

.gallery-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.gallery-header h3 {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
}

.close-btn {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: white;
  font-size: 24px;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

.gallery-content {
  padding: 20px;
}

.main-image-container {
  position: relative;
  margin-bottom: 16px;
}

#main-image {
  width: 100%;
  height: 300px;
  object-fit: cover;
  border-radius: 12px;
  filter: brightness(1.1) contrast(1.05);
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
}

.nav-buttons {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: 100%;
  display: flex;
  justify-content: space-between;
  padding: 0 10px;
  pointer-events: none;
}

.nav-btn {
  background: rgba(255, 255, 255, 0.9);
  border: none;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  font-size: 20px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  pointer-events: auto;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.nav-btn:hover {
  background: white;
  transform: scale(1.1);
}

.thumbnails {
  display: flex;
  gap: 8px;
  overflow-x: auto;
  padding: 8px 0;
}

.thumbnail {
  width: 60px;
  height: 60px;
  object-fit: cover;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  opacity: 0.6;
  border: 2px solid transparent;
}

.thumbnail:hover {
  opacity: 0.8;
  transform: scale(1.05);
}

.thumbnail.active {
  opacity: 1;
  border-color: #667eea;
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.3);
}

.image-counter {
  text-align: center;
  padding: 12px 20px;
  background: #f8f9fa;
  color: #666;
  font-size: 14px;
  font-weight: 500;
}

/* Responsive adjustments */
@media (max-width: 768px) {
  .gallery-container {
    max-width: 95%;
    max-height: 90%;
  }

  #main-image {
    height: 250px;
  }

  .gallery-header h3 {
    font-size: 16px;
  }

  .nav-btn {
    width: 35px;
    height: 35px;
    font-size: 18px;
  }
}
</style>