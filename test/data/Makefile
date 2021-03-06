
OSRM_RELEASE := $(shell node -e "console.log(require('../../package.json').osrm_release)")

ifeq ($(OSRM_RELEASE),)
$(error OSRM_RELEASE variable was not correct set)
endif

CAR_PROFILE_URL:=https://raw.githubusercontent.com/Project-OSRM/osrm-backend/$(OSRM_RELEASE)/profiles/car.lua
LIB_GUIDANCE_URL:=https://raw.githubusercontent.com/Project-OSRM/osrm-backend/$(OSRM_RELEASE)/profiles/lib/guidance.lua
PROFILE_LIB_ACCESS_URL:=https://raw.githubusercontent.com/Project-OSRM/osrm-backend/$(OSRM_RELEASE)/profiles/lib/access.lua
PROFILE_LIB_DESTINATION_URL:=https://raw.githubusercontent.com/Project-OSRM/osrm-backend/$(OSRM_RELEASE)/profiles/lib/destination.lua
PROFILE_LIB_SET_URL:=https://raw.githubusercontent.com/Project-OSRM/osrm-backend/$(OSRM_RELEASE)/profiles/lib/set.lua
PROFILE_LIB_SEQUENCE_URL:=https://raw.githubusercontent.com/Project-OSRM/osrm-backend/$(OSRM_RELEASE)/profiles/lib/sequence.lua
BERLIN_URL:=https://s3.amazonaws.com/mapbox/node-osrm/testing/berlin-latest.osm.pbf

all: berlin-latest.osrm.hsgr

lib/destination.lua:
	mkdir -p lib
	wget $(PROFILE_LIB_DESTINATION_URL) -O lib/destination.lua

lib/set.lua:
	mkdir -p lib
	wget $(PROFILE_LIB_SET_URL) -O lib/set.lua

lib/sequence.lua:
	mkdir -p lib
	wget $(PROFILE_LIB_SEQUENCE_URL) -O lib/sequence.lua

lib/access.lua:
	mkdir -p lib
	wget $(PROFILE_LIB_ACCESS_URL) -O lib/access.lua

lib/guidance.lua:
	wget $(LIB_GUIDANCE_URL) -O lib/guidance.lua

car.lua: lib/access.lua lib/destination.lua lib/guidance.lua lib/set.lua lib/sequence.lua
	wget $(CAR_PROFILE_URL) -O car.lua

clean:
	rm -f ./berlin-latest.*
	rm -f ./car.lua
	rm -rf ./lib

berlin-latest.osm.pbf:
	wget $(BERLIN_URL) -O berlin-latest.osm.pbf

berlin-latest.osrm: berlin-latest.osm.pbf car.lua $(OSRM_EXTRACT)
	@echo "Verifiyng data file integrity..."
	node md5sum.js -c data.md5sum
	@PATH="../../lib/binding:${PATH}" && echo "*** Using osrm-extract from `which osrm-extract` ***"
	@echo "Running osrm-extract..."
	@PATH="../../lib/binding:${PATH}" && osrm-extract berlin-latest.osm.pbf -p car.lua

berlin-latest.osrm.hsgr: berlin-latest.osrm car.lua $(OSRM_CONTRACT)
	@PATH="../../lib/binding:${PATH}" && echo "*** Using osrm-contract from `which osrm-contract` ***"
	@echo "Running osrm-contract..."
	@PATH="../../lib/binding:${PATH}" && osrm-contract berlin-latest.osrm

checksum: car.lua lib/destination.lua lib/access.lua lib/guidance.lua lib/set.lua lib/sequence.lua berlin-latest.osm.pbf
	node md5sum.js $^ > data.md5sum

.PHONY: clean checksum
