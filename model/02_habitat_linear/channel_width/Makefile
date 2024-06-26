# --------
# calculate channel width - measured / mapped / modelled
# --------

.PHONY: all clean

# point psql to db and stop on errors
PSQL=psql $(DATABASE_URL) -v ON_ERROR_STOP=1

WSG = $(shell $(PSQL) -AtX -c "SELECT watershed_group_code FROM whse_basemapping.fwa_watershed_groups_poly ORDER BY watershed_group_code")

GENERATED_FILES = .make/bcdata \
	.make/channel_width_measured \
	.make/channel_width_mapped \
	channel_width_analysis.csv \
	.make/channel_width_modelled \
	.make/channel_width

GENERATED_TABLES =  whse_fish.fiss_stream_sample_sites_sp\
	bcfishpass.channel_width_measured \
	bcfishpass.channel_width_mapped \
	bcfishpass.channel_width_analysis \
	bcfishpass.channel_width_modelled \
	bcfishpass.channel_width

all: $(GENERATED_FILES)

# --------
# download required BC data
# --------
.make/bcdata:
	mkdir -p .make
	bcdata bc2pg WHSE_FISH.FISS_STREAM_SAMPLE_SITES_SP
	$(PSQL) -c "CREATE INDEX ON whse_fish.fiss_stream_sample_sites_sp (new_watershed_code)"
	bcdata bc2pg WHSE_TERRESTRIAL_ECOLOGY.ERC_ECOSECTIONS_SP
	bcdata bc2pg WHSE_FOREST_VEGETATION.BEC_BIOGEOCLIMATIC_POLY
	$(PSQL) -f sql/load_bec_eco.sql
	touch $@

# --------
# MEASURED
# --------
.make/channel_width_measured: .make/bcdata sql/channel_width_measured.sql sql/fiss_stream_sample_sites_events.sql
	# match stream sample sites to streams
	$(PSQL) -f sql/fiss_stream_sample_sites_events.sql
	# Now load the measured channel widths where we have them, averaging measurements on the same stream
	# NOTE - this presumes PSCIS data is already loaded
	$(PSQL) -f sql/channel_width_measured.sql
	touch $@

# --------
# MAPPED
# --------
.make/channel_width_mapped:
	$(PSQL) -c "DROP TABLE IF EXISTS bcfishpass.channel_width_mapped;"
	$(PSQL) -c "CREATE TABLE bcfishpass.channel_width_mapped ( \
	  linear_feature_id bigint, \
	  watershed_group_code text, \
	  channel_width_mapped numeric, \
	  cw_stddev numeric, \
	  UNIQUE (linear_feature_id) );"
	# load each watershed group seperately, in parallel
	parallel $(PSQL) -f sql/channel_width_mapped.sql -v wsg={1} ::: $(WSG)
	$(PSQL) -c "CREATE INDEX ON bcfishpass.channel_width_mapped (linear_feature_id)"
	touch $@

# --------
# REPORT - report on measured and mapped data points for generating the model
# --------
channel_width_analysis.csv: .make/channel_width_mapped .make/channel_width_measured
	$(PSQL) -f sql/channel_width_analysis.sql
	psql2csv $(DATABASE_URL) "SELECT * FROM bcfishpass.channel_width_analysis" > channel_width_analysis.csv

# --------
# MODELLED
# --------
.make/channel_width_modelled: .make/channel_width_measured .make/channel_width_mapped sql/channel_width_modelled.sql
	# run the model as a single query, it doesn't take too long to process
	$(PSQL) -f sql/channel_width_modelled.sql
	touch $@

# --------
# OUTPUT TABLE
# --------
# combine the measured/mapped/modelled data into a single table for easy relating to streams table
.make/channel_width: .make/channel_width_modelled sql/channel_width.sql
	$(PSQL) -c "DROP TABLE IF EXISTS bcfishpass.channel_width; \
	CREATE TABLE bcfishpass.channel_width ( \
		linear_feature_id bigint primary key, \
	  channel_width_source text, \
	  channel_width double precision);"
	parallel $(PSQL) -f sql/channel_width.sql -v wsg={1} ::: $(WSG)
	touch $@

clean:
	rm -f .make
	# drop all target tables
	for table in $(GENERATED_TABLES); do \
		$(PSQL) -c "DROP TABLE IF EXISTS "$$table ;\
	done
