# JQ cheat sheet

Add new field into objects that is processed by other tool
```bash
while read l; do
	{ echo $l |
		jq '.n + .s' |
		tr '[A-Z]' '[a-z]' |
		jq '{u:.}';
	echo $l; } |
	jq -cn 'inputs + inputs';
done <names.json >names_w_nick.json
```
