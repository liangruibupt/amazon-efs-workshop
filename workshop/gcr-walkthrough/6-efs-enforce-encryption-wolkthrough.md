# Detecting Files Systems That Are Unencrypted at Rest
## Create a Metric Filter
{ ($.eventName = CreateFileSystem) && ($.responseElements.encrypted IS FALSE) } 