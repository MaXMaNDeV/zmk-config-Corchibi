# Patch 1: Skip CONFIGURATION_RESET on startup (causes I2C issues on nRF52840)
# 蓄積フィルタは削除: k_work_submitループによるバッテリー消耗を回避
# オートマウス誤発動防止は require-prior-idle-ms=500 で対応

set(FILE_PATH "${ZEPHYR_BASE}/drivers/input/input_pat912x.c")
file(READ "${FILE_PATH}" FILE_CONTENT)

# Patch 1: Skip CONFIGURATION_RESET write
string(
  REPLACE "\ti2c_reg_write_byte_dt(&cfg->i2c, PAT912X_CONFIGURATION, CONFIGURATION_RESET);\n"
          "\t\/\/ i2c_reg_write_byte_dt(&cfg->i2c, PAT912X_CONFIGURATION, CONFIGURATION_RESET);\n"
  FILE_CONTENT "${FILE_CONTENT}"
)

file(WRITE "${FILE_PATH}" "${FILE_CONTENT}")
message(STATUS "Patched PAT912x driver (CONFIGURATION_RESET skip): ${FILE_PATH}")
