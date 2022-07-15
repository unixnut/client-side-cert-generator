# Targets (targets with "*" require the CLIENT variables on the command line):
#   all		Sets up CA
#   add*	Used by 'zip'
#   zip*	Used for making a client cert for a person's own computer
#   ctarball*	Used for making a client cert for a Linux host
#   revoke*	Revoke's a given cert and re-generates the CRL (don't forget to install it on the server)
#   tarball	For installation on the server
#   install_keys  Run on a Linux host after extracting the tarball; specify DEST
#   		  (Don't forget to install and modify the config file.)
#
# The following variables may/must be present on the command line:
#   CLIENT (if building a client; mandatory)
#   CLIENT_REQ_ARGS= (if building a client; used to require a passphrase)
#   KEY_DEPT (optional; specifies organizationalUnitName and alters config file name)
#   CN (optional; specifies commonName)
#   KEY_DIR (optional)
#   KEY_CONFIG (optional)
#   REQ_ARGS (optional; can be used to pass "-subj 'arg'" -- NOTE QUOTES)
#   CA_ARGS (optional; can be used to pass "-batch")
#
# The following environment variables may/must be present:
#   KEY_ORG (mandatory)
#   KEY_SIZE (optional; do not change after CA is built; edit server/staff.conf)
#
# TO-DO:
#   + add 'remove' target; runs "rm -rf keys/$(CLIENT).* clients/.$(CLIENT)_*stamp clients/$(CLIENT)"
#   + make 'remove' target depnd on 'revoke' target

# == Setup ==
.EXPORT_ALL_VARIABLES:

# must be a single word
CLIENT = someone
# don't use a passphrase
CLIENT_REQ_ARGS = -nodes

DEST = /tmp/keys

KEY_CONFIG = openssl.cnf
KEY_DIR = keys
# These might be overriden by an environment variable
KEY_SIZE ?= 2048
KEY_DEPT ?= 
CN ?= 

CLIENT_DEPS = clients/.$(CLIENT)_stamp


# *** TARGETS ***
.PHONY: all
# make sure all required files exist
all: $(KEY_DIR)/ca.key


# == Client ==
.PHONY: add cc copy_key conf_clean conf_link revoke
# this target generates the following in clients/:
#   + a TLS mode client config file called someone.conf
#   + a directory called someone containing:
#       - a symbolic link called $KEY_ORG.ovpn
#         (it will be called $KEY_DEPT.ovpn if $KEY_DEPT is set)
#       - a keys subdirectory containing symlinks to the client & ca certs and
#         the the client key
# Makes sure that if the .ovpn file is re-created, the zip/tarball will be too.
add: $(CLIENT_DEPS)

clients/$(CLIENT).conf: clients/_template.conf
	sed -e "s/<CLIENT>/$(CLIENT)/" $^ > $@

# -- directory containing links to be archived --
# Use stamp files rather than phony targets to avoid always rebuilding
clients/.$(CLIENT)_stamp: clients/$(CLIENT) \
 clients/.$(CLIENT)_cert-link-stamp
	touch $@

clients/$(CLIENT):
	mkdir -p $@ $@/keys

clients/.$(CLIENT)_cert-link-stamp: $(KEY_DIR)/$(CLIENT).crt \
 $(KEY_DIR)/$(CLIENT).key $(KEY_DIR)/$(CLIENT).pfx
	ln -sv --force ../../../$(KEY_DIR)/$(CLIENT).crt ../../../$(KEY_DIR)/$(CLIENT).key ../../../$(KEY_DIR)/$(CLIENT).pfx clients/$(CLIENT)/keys
	ln -s --force ../../../$(KEY_DIR)/ca.crt clients/$(CLIENT)/keys
	touch $@

# -- real .key and .crt files --
$(KEY_DIR)/$(CLIENT).crt: $(KEY_DIR)/$(CLIENT).csr $(KEY_DIR)/ca.crt $(KEY_DIR)/ca.key
	openssl ca -config $(KEY_CONFIG) $(CA_ARGS) \
	  -in $(KEY_DIR)/$(CLIENT).csr -out $(KEY_DIR)/$(CLIENT).crt

# -nodes isn't specified here because it might be in $(CLIENT_REQ_ARGS)
$(KEY_DIR)/$(CLIENT).key $(KEY_DIR)/$(CLIENT).csr: $(KEY_DIR)/ca.key
	openssl req -config $(KEY_CONFIG) $(REQ_ARGS) $(CLIENT_REQ_ARGS) \
	  -new -out $(KEY_DIR)/$(CLIENT).csr \
	  -newkey rsa:$(KEY_SIZE) -keyout $(KEY_DIR)/$(CLIENT).key
	chmod 0600 $(KEY_DIR)/$(CLIENT).key

$(KEY_DIR)/$(CLIENT).pfx: $(KEY_DIR)/$(CLIENT).key $(KEY_DIR)/$(CLIENT).crt
	openssl pkcs12 -export -password pass:'' -out $@ \
	  -inkey $(KEY_DIR)/$(CLIENT).key -in $(KEY_DIR)/$(CLIENT).crt

# -- revocation --
revoke:
	openssl ca -config $(KEY_CONFIG) -revoke "$(KEY_DIR)/$(CLIENT).crt"
	$(MAKE) tarball

# == Server ==
# updates the CRL whenever index.txt has changed
# (avoids having to generate the CRL every time a cert is revoked,
# but does take the safe option of making $(SERVER_ID).tar.gz transitively dependent
# on $(KEY_DIR)/index.txt)
$(KEY_DIR)/banned_certs.crl: $(KEY_DIR)/ca.key $(KEY_DIR)/index.txt
	[ -L $(KEY_DIR)/crl.pem ] || ln -s banned_certs.crl $(KEY_DIR)/crl.pem
	openssl ca -config $(KEY_CONFIG) \
	  -gencrl -out $(KEY_DIR)/banned_certs.crl


# == distribution archives ==
.phony: zip tarball ctarball
# this target creates a zipfile (no symlinks) for distribution to a client
zip: clientcert_$(CLIENT).zip
clientcert_$(CLIENT).zip: $(CLIENT_DEPS)
	# zip automatically dereferences symlinks
	(cd clients/$(CLIENT) ;  zip -r - . ) > $@

# this target creates a tarball (no symlinks) for distribution to a client machine
ctarball: clientcert_$(CLIENT).tar.gz
clientcert_$(CLIENT).tar.gz: $(CLIENT_DEPS)
	tar czhvf $@ -C clients/$(CLIENT) keys


# == CA ==
# Note: $(REQ_ARGS) is not used
$(KEY_DIR)/ca.crt: | $(KEY_DIR) $(KEY_DIR)/ca.key
$(KEY_DIR)/ca.key: | $(KEY_DIR) $(KEY_DIR)/index.txt $(KEY_DIR)/serial
	@echo Creating CA key and certificate
	openssl req -config $(KEY_CONFIG) \
	  -new -x509 -days 3650 -out $(KEY_DIR)/ca.crt \
	  -newkey rsa:$(KEY_SIZE) -keyout $(KEY_DIR)/ca.key -nodes
	chmod 0600 $(KEY_DIR)/ca.key

$(KEY_DIR):
	mkdir -p $(KEY_DIR)

$(KEY_DIR)/index.txt:
	touch $@

$(KEY_DIR)/serial:
	echo 01 > $@


# == Other ==
.PHONY: install_keys
# used to install keys as root for a specific client into a specfic directory
# TO-DO: install the config file too
install_keys:
	install -d "$(DEST)"
	install --mode=644 --preserve-timestamps "$(KEY_DIR)/$(CLIENT).crt" "$(KEY_DIR)/ca.crt" "$(DEST)"
	install --mode=600 --preserve-timestamps "$(KEY_DIR)/$(CLIENT).key" "$(DEST)"

